# SBT Arm -- IDA Pro Plugin (Static Binary Translation) by Luigi Origa

> **[!] This is a prototype / work-in-progress.**
> Features marked as *TODO* are planned but not available in the current build.

A **Static Binary Translation** plugin for **IDA Pro** that reads raw
**ARMv7-A (AArch32)** machine code from the IDA database and translates it
into compilable, standalone **C source code**.
It supports **ARM (A32)**, **Thumb (T16)**, and **Thumb-2 (T32)** instruction
sets with automatic mode detection from IDA's `T` segment register.
The generated C files can be compiled with any standard C compiler (GCC, Clang,
MSVC) and executed natively on the host machine, enabling offline analysis,
fuzzing, unit testing, and differential comparison of firmware functions.

---

## Table of Contents

- [Part 1 -- Usage Reference](#part-1----usage-reference)
  - [Keyboard Shortcuts](#keyboard-shortcuts)
  - [How It Works](#how-it-works)
  - [Output Files](#output-files)
  - [Translation Pipeline](#translation-pipeline)
  - [Configuration](#configuration)
  - [Optimization Levels](#optimization-levels)
  - [Memory Model](#memory-model)
  - [Known Library Functions](#known-library-functions)
  - [Fuzz Harness](#fuzz-harness)
  - [Batch Mode](#batch-mode)
  - [Notes](#notes)
- [Part 2 -- Supported Opcodes](#part-2----supported-opcodes)
  - [Opcode Categories 1-16](#1-thumb-16-bit)
  - [Summary](#summary-by-category)

---

# Part 1 -- Usage Reference

## Keyboard Shortcuts

| Shortcut | Action |
|----------|--------|
| `Alt-7`  | Run the static binary translator on the current function |

---

## How It Works

Place the cursor on any function in IDA's disassembly view and press **Alt-7**.
The plugin performs a multi-phase translation:

1. Crawls the function's call graph (BFS, up to 16 levels deep)
2. Resolves indirect jumps and calls via IDA cross-references
3. Classifies memory references into Flash, SRAM, and MMIO regions
4. Collects global variable aliases from IDA's name list
5. Generates optimized and non-optimized C translations, plus a header file

No comment-based commands are needed -- the plugin is fully automatic.

**Example output** (excerpt from an optimized translation):

```c
void sbt_my_function( uint32_t* r, uint32_t* cpsr ) {
    sbt_arm_reset_runtime();
    sbt_init_data();
    /* PUSH {r4-r7, lr} */
    sbt_mem_wr32( r[13] - 4,  r[14] );
    sbt_mem_wr32( r[13] - 8,  r[7] );
    ...
    if( cond_EQ() ) goto L_00008042;
    r[0] = r[4] + r[5];  /* ADD r0, r4, r5 */
    ...
}
```

---

## Output Files

For a function named `my_func`, the plugin generates three files in the same
directory as the IDB:

| File | Description |
|------|-------------|
| `my_func_sbt.c` | Optimized C translation (opt_level 5) |
| `my_func_sbt_noopt.c` | Non-optimized version (opt_level 0, raw goto + labels) |
| `my_func_sbt.h` | Header file with memory region externs, global variable `#define` macros, function prototypes, and `sbt_init_data()` |

The `.c` file is fully self-contained: it includes the ARM runtime support
library (flag computation, barrel shifter, condition evaluation, VFP/NEON
helpers) as inlined source code, so no external dependencies are needed beyond
a standard C compiler and `<stdint.h>`.

---

## Translation Pipeline

| Phase | Description |
|-------|-------------|
| **1** | **Call-graph crawl** -- BFS from cursor function through call targets and tail calls, up to `max_call_depth` (default 16) |
| **1.5** | **Indirect resolution** -- Resolve indirect jumps/calls via IDA xrefs |
| **2** | **Region classification** -- Classify absolute address references into Flash/ROM, SRAM, and MMIO/SFR using ARM Cortex-M address map heuristics |
| **2.5** | **Global aliases** -- Collect named global variables from IDA for `#define` macros in the `.h` file |
| **3** | **C generation (optimized)** -- Generate the `.c` file at opt_level 5 |
| **4** | **C generation (non-optimized)** -- Generate the `_noopt.c` file at opt_level 0 |
| **5** | **Header generation** -- Generate the `.h` file with externs, aliases, and prototypes |

---

## Configuration

Configuration is set at compile time via the `SbtConfig` struct defaults.
Current defaults:

| Parameter | Default | Description |
|-----------|---------|-------------|
| `emit_asm_comments` | `true` | Emit original ARM assembly as inline C comments |
| `emit_fuzz_harness` | `true` | Generate `main()` under `#ifdef SBT_FUZZ_HARNESS` |
| `emit_libfuzzer` | `true` | Generate `LLVMFuzzerTestOneInput` under `#ifdef SBT_LIBFUZZER` |
| `emit_addr_labels` | `false` | Emit `L_XXXXXXXX:` labels at every instruction |
| `opt_level` | `5` | Optimization level (0--5) |
| `mem_size` | `16 MiB` | Size of the flat memory buffer |
| `follow_calls` | `true` | Recursively translate called functions |
| `max_call_depth` | `16` | Maximum BFS depth for call-graph crawl |

---

## Optimization Levels

| Level | Description |
|-------|-------------|
| **0** | None -- raw `goto` + `L_XXXXXXXX:` labels, one statement per instruction |
| **1** | If-blocks -- negate forward gotos into `if` statements |
| **2** | + Loops -- `do-while`, `while`, `for` pattern recognition |
| **3** | + Control flow -- `if-else`, `break`, `continue` |
| **4** | + Expressions -- `&&`/`||` short-circuit, ternary `?:` |
| **5** | + Advanced -- `switch`, `return`, function inlining |

The optimized file (`_sbt.c`) uses level 5.
The non-optimized file (`_sbt_noopt.c`) uses level 0.

---

## Memory Model

The plugin supports two memory model variants depending on the IDA database:

### Flat Memory

When no IDA segments are available, a single flat buffer is used:

```c
static uint8_t sbt_mem[0x1000000];  /* 16 MiB */
```

All memory accesses are bounds-masked to fit within the buffer.

### Region-Based Memory

When IDA segments are available, the plugin creates separate buffers per segment
with address-range-based routing. Memory regions are classified using ARM
Cortex-M address map heuristics:

| Address Range | Classification |
|---------------|----------------|
| `< 0x20000000` | Flash / ROM (pre-initialized with IDB data) |
| `0x20000000 -- 0x3FFFFFFF` | SRAM |
| `0x40000000 -- 0x5FFFFFFF` | Peripheral / MMIO (stub functions) |
| `>= 0xE0000000` | System / PPB / vendor SFR (stub functions) |

Flash/ROM regions are pre-loaded with actual data from the IDA database via
`sbt_init_data()`. MMIO regions generate read/write stub functions that can
be customized by the user.

---

## Known Library Functions

The translator recognizes common C library functions and replaces them with
native C calls instead of translating their instruction bytes:

| Function | Replacement |
|----------|-------------|
| `memcpy` | `memcpy( sbt_ptr(r[0]), sbt_ptr(r[1]), r[2] )` |
| `memmove` | `memmove( sbt_ptr(r[0]), sbt_ptr(r[1]), r[2] )` |
| `memset` | `memset( sbt_ptr(r[0]), (int)r[1], r[2] )` |
| `memcmp` | `r[0] = memcmp( sbt_ptr(r[0]), sbt_ptr(r[1]), r[2] )` |
| `strlen` | `r[0] = strlen( sbt_ptr(r[0]) )` |
| `strcmp` | `r[0] = strcmp( sbt_ptr(r[0]), sbt_ptr(r[1]) )` |
| `strncmp` | `r[0] = strncmp( sbt_ptr(r[0]), sbt_ptr(r[1]), r[2] )` |
| `strcpy` | `strcpy( sbt_ptr(r[0]), sbt_ptr(r[1]) )` |
| `strncpy` | `strncpy( sbt_ptr(r[0]), sbt_ptr(r[1]), r[2] )` |
| `strcat` | `strcat( sbt_ptr(r[0]), sbt_ptr(r[1]) )` |
| `strchr` | `r[0] = strchr( sbt_ptr(r[0]), r[1] )` |
| `rand` | `r[0] = rand()` |
| `srand` | `srand( r[0] )` |
| `abs` | `r[0] = abs( r[0] )` |
| `malloc` | Stub (not supported in flat memory) |
| `free` | Stub (not supported in flat memory) |
| `calloc` | Stub (not supported in flat memory) |
| `realloc` | Stub (not supported in flat memory) |
| `printf` | Stub |
| `sprintf` | Stub |
| `snprintf` | Stub |

---

## Fuzz Harness

The generated `.c` file includes two optional entry points for fuzzing,
gated behind preprocessor macros:

### AFL / standalone harness

Compile with `-DSBT_FUZZ_HARNESS` to include a `main()` that reads
register inputs from `stdin`:

```bash
gcc -DSBT_FUZZ_HARNESS -o fuzz_target my_func_sbt.c
echo "AAAA" | ./fuzz_target
```

### libFuzzer harness

Compile with `-DSBT_LIBFUZZER` to include `LLVMFuzzerTestOneInput()`:

```bash
clang -fsanitize=fuzzer -DSBT_LIBFUZZER -o fuzz_target my_func_sbt.c
./fuzz_target corpus/
```

---

## Batch Mode

The plugin supports IDA batch mode (`-A` flag) for headless operation.
When IDA is running in batch mode, the plugin waits for auto-analysis to
complete, then automatically translates the function specified via a netnode
(`$ sbt_target_ea`).

---

## Notes

- This is a **prototype**: output correctness, completeness, and C code
  quality are not guaranteed for all ARM instruction sequences.
- ARM/Thumb mode is auto-detected from IDA's `T` segment register per
  instruction byte, allowing mixed-mode functions.
- IT block (If-Then) handling is fully implemented with proper CPSR flag
  preservation during conditional Thumb-2 sequences.
- All 14 ARM condition codes are supported via inline macros.
- Full NZCV flag tracking for arithmetic and logical operations.
- Callee-saved registers (`r4`--`r11`) are preserved around inter-function calls.
- The generated code uses the **AAPCS32** calling convention (arguments in
  `r[0]`--`r[3]`, return value in `r[0]`).
- Exclusive load/store instructions (`LDREX`/`STREX`) include an exclusive
  monitor simulation.
- Feedback and bug reports are welcome via the issue tracker.

---

# Part 2 -- Supported Opcodes

## 1. Thumb (16-bit)

| Mnemonic | Description |
|----------|-------------|
| LSL / LSR / ASR | Shift immediate |
| ADD / SUB | Register and 3-bit/8-bit immediate |
| MOV / CMP | Immediate (8-bit) |
| AND / EOR / ADC / SBC / ROR / TST / RSB / CMN / ORR / MUL / BIC / MVN | Data processing (register) |
| ADD / CMP / MOV (high) | High register operations (r8--r15) |
| BX / BLX | Branch and exchange (register) |
| LDR (literal) | PC-relative load |
| LDR / STR / LDRB / STRB / LDRH / STRH | Load/store with immediate and register offsets |
| LDR / STR (SP) | SP-relative load/store |
| ADR | PC + immediate |
| ADD / SUB (SP) | Stack pointer adjustment |
| PUSH / POP | Push/pop with LR/PC |
| STM / LDM | Store/load multiple |
| B.cond | Conditional branch |
| B | Unconditional branch |
| CBZ / CBNZ | Compare and branch if zero/not zero |
| SXTH / SXTB / UXTH / UXTB | Sign/zero extend |
| REV / REV16 / REVSH | Byte reversal |
| IT | If-Then block |
| BKPT | Breakpoint |
| NOP / WFE / WFI / SEV / YIELD | Hints |

## 2. Thumb-2 (32-bit)

| Mnemonic | Description |
|----------|-------------|
| BL / B.W | Branch with link, wide conditional/unconditional |
| BLX (imm) | Branch with link and exchange |
| AND / BIC / ORR / ORN / EOR / ADD / ADC / SBC / SUB / RSB | Data processing (modified immediate) |
| MOV / MVN / TEQ / TST / CMP / CMN | Data processing (modified immediate, flags only) |
| AND / BIC / ORR / ORN / EOR / ADD / ADC / SBC / SUB / RSB | Data processing (shifted register) |
| MOVW / MOVT | 16-bit wide immediate |
| ADDW / SUBW | 12-bit wide immediate |
| LDR / STR / LDRB / STRB / LDRH / STRH | 12-bit immediate offset |
| LDR / STR (signed offset) | 8-bit signed offset, pre/post-index |
| LDR (literal) | PC-relative |
| LDRD / STRD | Double register load/store |
| LDM.W / STM.W / STMDB | Wide load/store multiple |
| PUSH.W / POP.W | Wide push/pop |
| TBB / TBH | Table branch byte/halfword |
| BFI / BFC / SBFX / UBFX | Bit field manipulation |
| PKHBT / PKHTB | Pack halfword |
| MUL / MLA / MLS | Multiply / multiply-accumulate |
| SDIV / UDIV | Signed/unsigned divide |
| UMULL / UMLAL / SMULL / SMLAL / UMAAL | Long multiply |
| SXTH.W / SXTB.W / UXTH.W / UXTB.W | Wide sign/zero extend |
| SXTAB / SXTAH / UXTAB / UXTAH | Extend with add |
| CLZ | Count leading zeros |
| MRS / MSR | Status register access |
| CLREX / DMB / DSB / ISB | Barriers and exclusive clear |

## 3. ARM (A32) -- Data Processing

| Mnemonic | Description |
|----------|-------------|
| AND | Bitwise AND |
| EOR | Bitwise Exclusive OR |
| SUB | Subtract |
| RSB | Reverse subtract |
| ADD | Add |
| ADC | Add with carry |
| SBC | Subtract with carry |
| RSC | Reverse subtract with carry |
| TST | Test (AND, flags only) |
| TEQ | Test equivalence (EOR, flags only) |
| CMP | Compare (SUB, flags only) |
| CMN | Compare negative (ADD, flags only) |
| ORR | Bitwise OR |
| MOV | Move |
| BIC | Bit clear |
| MVN | Move NOT |
| MOVW | Move wide (16-bit immediate) |
| MOVT | Move top (16-bit to upper halfword) |

## 4. ARM (A32) -- Multiply

| Mnemonic | Description |
|----------|-------------|
| MUL | Multiply (32x32->32) |
| MLA | Multiply-accumulate |
| MLS | Multiply-subtract |
| UMULL | Unsigned multiply long (32x32->64) |
| UMLAL | Unsigned multiply-accumulate long |
| SMULL | Signed multiply long (32x32->64) |
| SMLAL | Signed multiply-accumulate long |
| UMAAL | Unsigned multiply-accumulate-accumulate long |

## 5. ARM (A32) -- Load / Store

| Mnemonic | Description |
|----------|-------------|
| LDR / STR | Load/store word (immediate and register offset, pre/post-index) |
| LDRB / STRB | Load/store byte |
| LDRH / STRH | Load/store halfword |
| LDRSB / LDRSH | Load signed byte/halfword |
| LDRD / STRD | Load/store doubleword |
| LDREX / STREX | Load/store exclusive (word) |
| LDREXB / STREXB | Load/store exclusive (byte) |
| LDREXH / STREXH | Load/store exclusive (halfword) |
| LDREXD / STREXD | Load/store exclusive (doubleword) |
| LDM / STM | Load/store multiple (IA/IB/DA/DB) |
| PUSH / POP | Push/pop (aliases for STM/LDM) |
| SWP / SWPB | Swap (deprecated) |
| PLD / PLI | Preload hints |

## 6. ARM (A32) -- Branch

| Mnemonic | Description |
|----------|-------------|
| B | Branch |
| BL | Branch with link |
| BX | Branch and exchange |
| BLX | Branch with link and exchange |

## 7. ARM (A32) -- Status Register

| Mnemonic | Description |
|----------|-------------|
| MRS | Move from status register |
| MSR | Move to status register |

## 8. ARM (A32) -- Saturating / Bit Field / Packing

| Mnemonic | Description |
|----------|-------------|
| QADD / QSUB / QDADD / QDSUB | Saturating arithmetic |
| SSAT / USAT / SSAT16 / USAT16 | Signed/unsigned saturate |
| BFI / BFC | Bit field insert/clear |
| SBFX / UBFX | Bit field extract |
| PKHBT / PKHTB | Pack halfword |
| SEL | Select bytes |
| CLZ | Count leading zeros |
| RBIT / REV / REV16 / REVSH | Bit/byte reversal |

## 9. ARM (A32) -- DSP / SIMD

| Mnemonic | Description |
|----------|-------------|
| SADD16 / SSUB16 / SADD8 / SSUB8 | Signed parallel add/subtract |
| UADD16 / USUB16 / UADD8 / USUB8 | Unsigned parallel add/subtract |
| QADD16 / QSUB16 / QADD8 / QSUB8 | Saturating parallel |
| SHADD16 / SHSUB16 / SHADD8 / SHSUB8 | Halving parallel |
| UHADD16 / UHSUB16 / UHADD8 / UHSUB8 | Unsigned halving parallel |
| SMLABB / SMLABT / SMLATB / SMLATT | Signed halfword multiply-accumulate |
| SMLAWB / SMLAWT / SMULWB / SMULWT | Signed word-by-halfword multiply |
| SMLAD / SMLADX / SMLSD / SMLSDX | Dual multiply-add/subtract |
| SMLALD / SMLALDX / SMLSLD / SMLSLDX | Dual multiply-add/subtract long |
| SMMUL / SMMULR / SMMLA / SMMLAR / SMMLS / SMMLSR | Most-significant-word multiply |
| USAD8 / USADA8 | Unsigned sum of absolute differences |
| SXTAB16 / UXTAB16 | Extend byte16 with add |

## 10. ARM (A32) -- Divide

| Mnemonic | Description |
|----------|-------------|
| SDIV | Signed divide |
| UDIV | Unsigned divide |

## 11. VFP (Floating-Point)

| Mnemonic | Description |
|----------|-------------|
| VLDR / VSTR | Load/store FP register (F32 / F64) |
| VADD / VSUB / VMUL / VDIV | FP arithmetic (F32 / F64) |
| VNEG / VABS / VSQRT | FP unary (F32 / F64) |
| VMLA / VMLS | FP multiply-accumulate (F32 / F64) |
| VNMLA / VNMLS / VNMUL | FP negated multiply (F32 / F64) |
| VMOV | Register, immediate, ARM core <-> VFP |
| VCMP | FP compare with FPSCR update (F32 / F64) |
| VCVT | Integer <-> float, single <-> double |
| VMRS / VMSR | FPSCR transfer |
| VPUSH / VPOP | Push/pop FP registers |
| VLDM / VSTM | Load/store multiple FP registers |

## 12. NEON (SIMD)

| Mnemonic | Description |
|----------|-------------|
| VADD.I / VSUB.I | Integer add/subtract (multi-size) |
| VMUL.I | Integer multiply |
| VAND / VBIC / VORR / VORN / VEOR | Bitwise logical |
| VMVN | Bitwise NOT |
| VSHL / VSHR | Immediate shift |
| VDUP.32 | Duplicate scalar |
| VADD.F32 / VSUB.F32 / VMUL.F32 | Float NEON lanes |

## 13. System / Hints

| Mnemonic | Description |
|----------|-------------|
| SVC | Supervisor call |
| CLREX | Clear exclusive |
| DMB / DSB / ISB | Memory barriers |
| NOP / WFE / WFI / SEV / YIELD | Hints |

---

## Summary by Category

| #  | Category                         | Count |
|----|----------------------------------|-------|
| 1  | Thumb (16-bit)                   | 40+   |
| 2  | Thumb-2 (32-bit)                 | 50+   |
| 3  | ARM Data Processing              | 18    |
| 4  | ARM Multiply                     | 8     |
| 5  | ARM Load / Store                 | 20+   |
| 6  | ARM Branch                       | 4     |
| 7  | ARM Status Register              | 2     |
| 8  | ARM Saturating/BitField/Packing  | 16    |
| 9  | ARM DSP / SIMD                   | 30+   |
| 10 | ARM Divide                       | 2     |
| 11 | VFP (Floating-Point)             | 20+   |
| 12 | NEON (SIMD)                      | 10+   |
| 13 | System / Hints                   | 8     |
|    | **TOTAL**                        |**~230+**|

---

## Notes

- **ARM/Thumb auto-detection** from IDA's `T` segment register (per-byte mode tracking).
- **IT block support** with full If-Then state machine and CPSR flag preservation.
- **Barrel shifter** with all shift modes (LSL, LSR, ASR, ROR) for immediate and register operands.
- **Condition codes**: all 14 ARM conditions supported via inline macros.
- **Exclusive monitor** simulation for `LDREX`/`STREX` instructions.
- **Full NZCV flag tracking** for arithmetic, logic, compare, and barrel shifter carry-out.
- **Callee-saved register preservation** for inter-function calls (r4--r11).
