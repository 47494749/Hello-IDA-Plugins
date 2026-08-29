# ARM Simulator -- IDA Pro Plugin (Prototype) by Luigi Origa

> **[!] This is a prototype / work-in-progress.**
> Features marked as *TODO* are planned but not available in the current build.

A lightweight **ARMv7-A (AArch32)** CPU simulator embedded inside **IDA Pro**.
It supports both **ARM (A32)** and **Thumb (T16/T32)** instruction sets, with
automatic mode detection from IDA's `T` segment register.
It allows you to simulate individual functions directly from the disassembly view,
inspect register and memory state, intercept memory reads with mock values, and
perform basic fuzzing over register and memory inputs -- all driven by structured
commands written as **repeatable comments at the top of the target function**.

---

## Table of Contents

- [Part 1 -- Command Reference](#part-1----command-reference)
  - [Keyboard Shortcuts](#keyboard-shortcuts)
  - [How It Works](#how-it-works)
  - [Command Syntax](#command-syntax)
  - [Register Assignment](#register-assignment)
  - [Simulation Commands](#simulation-commands)
  - [Memory Write](#memory-write)
  - [Memory Dump](#memory-dump)
  - [Mock Reads](#mock-reads)
  - [Save Flags](#save-flags)
  - [Generators](#generators)
  - [Fuzz Block](#fuzz-block)
  - [Notes](#notes)
- [Part 2 -- Supported Opcodes](#part-2----supported-opcodes)
  - [Opcode Categories 1-16](#1-data-processing)
  - [Summary](#summary-by-category)

---

# Part 1 -- Command Reference

## Keyboard Shortcuts

| Shortcut | Action |
|----------|--------|
| `Alt-1`  | Run the simulator on the current function |
| `Ctrl-1` | Show the command reference help |

---

## How It Works

Commands are placed as a **repeatable comment** (`;` in IDA's assembly view) at the very beginning of the function you want to simulate. The plugin parses these comments before execution begins.

```asm
; @r0 = 0x10
; @r1 = rand(0, 0xFF)
; @sp = stack(0)
; @max 5000
; @run
sub_8001234:
    ...
```

The simulator reads the comment block, sets up the CPU state accordingly, and begins emulation. Output (register dumps, memory dumps, trace files) is written to the IDA output window and/or to disk depending on the save flags used.

The simulator automatically detects ARM vs Thumb mode from IDA's `T` segment register and handles IT blocks, barrel shifter operations, and all 15 ARM condition codes.

---

## Command Syntax

- Commands begin with `@` followed by a keyword.
- Values can be **decimal** (`123`) or **hexadecimal** (`0x7B`).
- Lines starting with `;` or `#` inside the comment block are ignored (pure comments).
- Commands execute **sequentially** (pipeline order) unless they are save flags, which are always deferred to the end.
- Commands are **case-sensitive** (`@run` != `@Run`).
- If no `@run` is present, an **implicit** `@run` is inserted before deferred saves.
- Parse error on any line aborts the entire command pipeline.

---

## Register Assignment

Set the initial CPU state before simulation starts. All registers support a **value** or a **generator expression** (see [Generators](#generators)).

```
@r0..@r15  = value|gen     ; Set general-purpose register R0--R15
@sp        = value|gen     ; Alias for R13 (Stack Pointer)
@lr        = value|gen     ; Alias for R14 (Link Register)
@pc        = value         ; Alias for R15 (Program Counter)
@cpsr      = value|gen     ; Set Current Program Status Register
@spsr      = value|gen     ; Set Saved Program Status Register
@fpscr     = value|gen     ; Set Floating-Point Status/Control Register
@s0..@s31  = value|gen     ; Set VFP single-precision register
@d0..@d31  = value|gen     ; Set VFP double-precision register
@q0..@q15  = value|gen     ; Set NEON quadword register
```

**Examples:**

```
; @r0  = 0
; @r1  = rand(0, 0xFFFF)
; @sp  = 0x20001000
; @cpsr = flag(N=0, Z=0, C=0, V=0)
; @pc  = 0x08001000
```

---

## Simulation Commands

| Command | Description |
|---------|-------------|
| `@run` | Run or resume the simulator |
| `@debug` | Enable per-instruction debug log |
| `@max <N>` | Maximum number of instructions to execute (default 100000) |
| `@timeout <ms>` | Simulation timeout in milliseconds (default 30000) |
| `@loop_limit <N>` | Infinite-loop detection check interval (default 1000) |

Multiple `@run` commands in sequence resume emulation from the current PC.

---

## Breakpoints

| Command | Description |
|---------|-------------|
| `@bp <addr>` | Breakpoint at the given PC address |
| `@bp Rn op value` | Break when register `Rn` satisfies condition |
| `@bp memN[addr] op val` | Break on memory condition (`N` = 8, 16, 32, 64) |

Supported operators: `==`  `!=`  `>`  `<`  `>=`  `<=`

Breakpoints accumulate across multiple `@run` invocations.

**Examples:**

```
; @bp 0x08003FFC
; @bp R5 >= 0x100
; @bp mem32[0x20002000] != 0
; @max 10000
; @run
```

---

## Memory Write

Write values to memory before simulation starts. Supports single writes, list writes, and range fills.

```
@memN [addr] = value|gen          ; Write a single N-bit value
@memN [addr] = list(v1, v2, ...)  ; Write a sequence of N-bit values
@memN [begin-end] = value|gen     ; Fill a range with N-bit value
```

`N` must be one of: `8`, `16`, `32`, `64`.

**Examples:**

```
; @mem32 [0x20001000] = 0xDEADBEEF
; @mem8  [0x20002000] = list(0x01, 0x02, 0x03, 0xFF)
; @mem16 [0x20003000-0x20003020] = 0x0000
; @mem32 [0x20004000] = rand(0, 0xFFFFFFFF)
```

---

## Memory Dump

Dump memory contents to the IDA output window after simulation. Optionally save a binary snapshot to disk.

```
@dump8  (addr, len)               ; Dump len bytes as 8-bit hex
@dump16 (addr, len)               ; Dump len/2 words as 16-bit hex
@dump32 (addr, len)               ; Dump len/4 dwords as 32-bit hex
@dump64 (addr, len)               ; Dump len/8 qwords as 64-bit hex
@save_memory("path", addr, size)  ; Save memory region to binary file
```

**Examples:**

```
; @dump32(0x20001000, 64)
; @save_memory("C:/traces/heap_after.bin", 0x20001000, 0x200)
```

---

## Mock Reads

Intercept memory reads during simulation and return controlled values instead of actual memory content. Useful for simulating hardware registers, sensor inputs, or external bus responses.

```
@mock_readN addr = value|gen [, modifier]
```

`N` must be one of: `8`, `16`, `32`, `64`.

Multiple lines for the same `addr` + `N` define a **sequential phase list**: the simulator returns each value in order. The **modifier** controls what happens after the list is exhausted:

| Modifier | Behavior |
|----------|----------|
| *(none / `last`)* | Stay on the last value after exhaustion |
| `loop` | Loop from index 0 indefinitely |
| `loop(N)` | Loop from index `N` indefinitely |
| `loop(N, cnt)` | Loop from index `N` exactly `cnt` times, then stay on last |

**Examples:**

```
; @mock_read32 0xE000E010 = 0xCAFEBABE
; @mock_read8  0x40000020 = list(0x00, 0x01, 0x02), loop
; @mock_read16 0x40000030 = rand(0, 0xFFFF)
; @mock_read32 0x40000040 = list(10, 20, 30), loop(1, 5)
; @run
```

---

## Save Flags

Save flags are always executed at the very end of the pipeline, regardless of where they appear in the comment block.

| Flag | Description |
|------|-------------|
| `@save_trace` | Save the full instruction execution trace to file |
| `@save_memory_trace` | Save the memory access trace (reads and writes) to file |
| `@save_memory_map` | Save the memory access heat map to file |
| `@save_fuzz_results` | Save the results of all fuzz iterations to file |

---

## Generators

Generators are expressions that produce values dynamically. They can be used wherever a plain numeric value is accepted (register assignments, memory writes, mock reads, fuzz steps), unless noted otherwise.

### Group 1 -- Scalars *(registers, memory, fuzz)*

| Generator | Description |
|-----------|-------------|
| `<numeric>` | Plain literal value (decimal or `0x` hex) |
| `rand(min, max)` | Random value uniformly drawn from `[min, max]` |
| `list(v1, v2, ...)` | Single-shot: uses first value; fuzz mode: iterates through values |
| `const(val)` | Explicit constant value |

### Group 2 -- Register-relative *(registers, memory, fuzz)*

| Generator | Description |
|-----------|-------------|
| `neg(Rn)` | Arithmetic negation of the register value |
| `mirror(Rn)` | Bit-reversal of the register value |
| `arith(Rn, op, val)` | Arithmetic on register: `+` `-` `*` `/` `&` `\|` `^` `~` `%` `L`(shl) `R`(shr) `A`(asr) `a`(abs) `z`(clz) |
| `off(Rn, val)` | Register value plus a signed offset |
| `aligned(Rn, n)` | Register aligned down to `n` bytes |
| `overflow(Rn)` | Register + 0x7FFFFFFF |
| `underflow(Rn)` | Register - 0x80000000 |

### op_* shortcuts *(named operators, equivalent to arith)*

| Generator | Description |
|-----------|-------------|
| `op_add(reg, imm)` | `reg + imm` |
| `op_sub(reg, imm)` | `reg - imm` |
| `op_mul(reg, imm)` | `reg * imm` |
| `op_div(reg, imm)` | `reg / imm` (unsigned) |
| `op_mod(reg, imm)` | `reg % imm` |
| `op_and(reg, imm)` | `reg & imm` |
| `op_or(reg, imm)` | `reg \| imm` |
| `op_xor(reg, imm)` | `reg ^ imm` |
| `op_not(reg)` | `~reg` (bitwise complement) |
| `op_shl(reg, imm)` | `reg << imm` (logical left shift) |
| `op_shr(reg, imm)` | `reg >> imm` (logical right shift) |
| `op_asl(reg, imm)` | `reg << imm` (same as shl) |
| `op_asr(reg, imm)` | `reg >> imm` (arithmetic, sign-extended) |
| `op_abs(reg)` | Absolute value |
| `op_clz(reg)` | Count leading zeros |

### Group 3 -- Pointer *(registers, memory, fuzz)*

| Generator | Description |
|-----------|-------------|
| `stack(offset)` | Stack pointer (`R13`) plus a signed offset |

### Group 4 -- Buffer/structured data *(memory only, TODO -- parsed but not yet evaluated)*

| Generator | Description |
|-----------|-------------|
| `buf(size)` | Allocate buffer |
| `struct(hex)` | Raw hex struct data |
| `file(path)` | Load data from file |
| `template(name)` | Named template |
| `repeat(val, size)` | Repeated byte pattern |
| `mutate(file, n)` | Mutate file data N times |

### Group 5 -- Iteration-based *(fuzz only)*

| Generator | Description |
|-----------|-------------|
| `range(min, max [,step])` | Iterate from `min` to `max` by `step` (default 1) |
| `edges` | Boundary values: 0, 1, 0x7F, 0x80, 0xFE, 0xFF |
| `signed_edges` | Signed boundaries: -1, -128, -32768, -2^31, 0, 127, 32767 |
| `pow2` | Powers of 2: 1, 2, 4, ..., 2^31 |
| `pow2_minus1` | Powers of 2 minus 1: 0, 1, 3, 7, ..., 0xFFFFFFFF |
| `rot_walk` | Rotating single-bit walk |
| `byte_walk` | Walking byte: 0xFF, 0xFF00, 0xFF0000, 0xFF000000 |
| `walk(val)` | Walking bit XOR with val: val ^ (1 << (it%32)) |
| `flip(addr, n)` | Bit-flip: addr ^ (1 << (it%n)) |
| `freq(val, n)` | Repeated value N times |
| `prev_result` | Previous run result |
| `ret_val` | Return value from previous call |
| `state_from(Rn)` | *(TODO)* State snapshot from register |

### Special

| Generator | Description |
|-----------|-------------|
| `flag(N=v, Z=v, ...)` | Compose a CPSR value from individual flag assignments |

**Flag names:** `N`, `Z`, `C`, `V`, `Q`

**Example:**

```
; @cpsr = flag(N=0, Z=0, C=0, V=0)
```

---

## Fuzz Block

The fuzz block lets you run the function repeatedly while varying one or more inputs across a defined space. Each fuzz step re-initializes CPU state from the register assignments above, applies the fuzz target value, and runs the simulation.

```
@fuzz
  seed(N)                    ; set random seed
  sequence:                  ; (TODO) enable multi-step sequence mode
  parallel=N                 ; (TODO) number of parallel workers
  resume(file)               ; (TODO) resume from saved state
  <iterations> [call(addr)] TARGET = gen [stop=cond] [log=type] [options...]
```

### Targets

| Target | Description |
|--------|-------------|
| `R0`..`R15` | General-purpose register |
| `SP`, `LR` | Stack pointer / Link register |
| `CPSR` | Current program status register |
| `SP[offset]` | Stack memory at `SP + offset` |
| `mem8[addr]` .. `mem64[addr]` | Memory location (8/16/32/64-bit) |

### Stop Conditions

| Condition | Description |
|-----------|-------------|
| `crash` | Stop on any crash/trap |
| `any_exception` | Stop on any exception |
| `pc(addr)` | Stop when PC reaches address |
| `Rn==val` | Stop when register equals value |
| `Rn!=val` | Stop when register differs |
| `reg(Rn, val)` | *(TODO)* Stop when register equals value |
| `mem_write(addr)` | *(TODO)* Stop on write to address |

### Log Types *(TODO -- parsed but not yet evaluated at runtime)*

| Type | Description |
|------|-------------|
| `all` | Log all iterations |
| `crash_only` | Log only crashes |
| `coverage` | Log coverage data |
| `unique_paths` | Log unique execution paths |
| `retval` | Log return values |
| `file(name.csv)` | Log to specific file |

**Example:**

```
; @r0  = 0x20001000
; @r1  = 0x20001000
; @max 2000
; @fuzz
;   0 R1 = range(0, 0xFF, 1)
;   1 R1 = rand(0, 0xFFFFFFFF)
;   2 mem32[0x20001000] = rand(0, 0xFFFFFFFF)
; @save_fuzz_results
```

---

## Notes

- This is a **prototype**: stability, accuracy, and feature completeness are not guaranteed.
- ARM/Thumb mode is auto-detected from IDA's `T` segment register.
- The simulator supports both little-endian and big-endian (detected from IDA database).
- Call-depth tracking with shadow LR stack for BL/BLX subroutine calls.
- 16 MiB flat memory buffer with configurable base address.
- Feedback and bug reports are welcome via the issue tracker.

---

# Part 2 -- Supported Opcodes

## 1. Data Processing

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
| LSL | Logical shift left |
| LSR | Logical shift right |
| ASR | Arithmetic shift right |
| ROR | Rotate right |
| RRX | Rotate right extended |

## 2. Multiply

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

## 3. Load / Store (Single)

| Mnemonic | Description |
|----------|-------------|
| LDR | Load register (word) |
| LDRB | Load register (byte) |
| LDRH | Load register (halfword) |
| LDRSB | Load register signed (byte) |
| LDRSH | Load register signed (halfword) |
| LDRD | Load register (doubleword) |
| LDREX | Load exclusive (word) |
| LDREXB | Load exclusive (byte) |
| LDREXH | Load exclusive (halfword) |
| LDREXD | Load exclusive (doubleword) |
| STR | Store register (word) |
| STRB | Store register (byte) |
| STRH | Store register (halfword) |
| STRD | Store register (doubleword) |
| STREX | Store exclusive (word) |
| STREXB | Store exclusive (byte) |
| STREXH | Store exclusive (halfword) |
| STREXD | Store exclusive (doubleword) |
| PLD | Preload data |
| PLI | Preload instruction |

## 4. Load / Store Multiple

| Mnemonic | Description |
|----------|-------------|
| LDM | Load multiple (IA/IB/DA/DB) |
| STM | Store multiple (IA/IB/DA/DB) |
| PUSH | Push registers (alias for STMDB SP!) |
| POP | Pop registers (alias for LDMIA SP!) |

## 5. Branch

| Mnemonic | Description |
|----------|-------------|
| B | Branch |
| BL | Branch with link |
| BX | Branch and exchange |
| BLX | Branch with link and exchange |
| BXJ | Branch and exchange Jazelle |
| CBZ | Compare and branch if zero (Thumb) |
| CBNZ | Compare and branch if not zero (Thumb) |
| TBB | Table branch byte (Thumb) |
| TBH | Table branch halfword (Thumb) |
| IT | If-Then block (Thumb-2) |

## 6. Status Register Access

| Mnemonic | Description |
|----------|-------------|
| MRS | Move from status register |
| MSR | Move to status register |
| CPS | Change processor state |

## 7. VFP / NEON Coprocessor

| Mnemonic | Description |
|----------|-------------|
| VMOV | Move (immediate, register, GP<->FP) |
| VMRS | Move FP status to ARM register |
| VMSR | Move ARM register to FP status |
| VADD.F32/F64 | Floating-point add |
| VSUB.F32/F64 | Floating-point subtract |
| VMUL.F32/F64 | Floating-point multiply |
| VDIV.F32/F64 | Floating-point divide |
| VNEG.F32/F64 | Floating-point negate |
| VABS.F32/F64 | Floating-point absolute |
| VSQRT.F32/F64 | Floating-point square root |
| VCMP.F32/F64 | Floating-point compare |
| VMLA.F32/F64 | FP multiply-accumulate |
| VMLS.F32/F64 | FP multiply-subtract |
| VNMLA.F32/F64 | FP negated multiply-accumulate |
| VNMLS.F32/F64 | FP negated multiply-subtract |
| VNMUL.F32/F64 | FP negated multiply |
| VCVT | Convert (F64<->F32, FP<->integer) |
| VLDR | Load FP register |
| VSTR | Store FP register |
| VLDM | Load multiple FP registers |
| VSTM | Store multiple FP registers |
| VPUSH | Push FP registers |
| VPOP | Pop FP registers |
| VADD.I8/I16/I32/I64 | NEON integer add |
| VSUB.I8/I16/I32/I64 | NEON integer subtract |
| VMUL.I8/I16/I32 | NEON integer multiply |
| VAND | NEON bitwise AND |
| VORR | NEON bitwise OR |
| VEOR | NEON bitwise XOR |
| VBIC | NEON bitwise bit clear |
| VORN | NEON bitwise OR NOT |
| VMVN | NEON bitwise NOT |
| VSHL | NEON shift left |
| VSHR.S/U | NEON shift right (signed/unsigned) |
| VDUP | NEON duplicate scalar |

## 8. Miscellaneous / Hints

| Mnemonic | Description |
|----------|-------------|
| NOP | No operation |
| WFI | Wait for interrupt |
| WFE | Wait for event |
| SEV | Send event |
| YIELD | Yield |
| DBG | Debug hint |
| DMB | Data memory barrier |
| DSB | Data synchronization barrier |
| ISB | Instruction synchronization barrier |
| CLREX | Clear exclusive |
| BKPT | Breakpoint |
| SWP/SWPB | Swap (deprecated) |

## 9. Saturating Arithmetic

| Mnemonic | Description |
|----------|-------------|
| QADD | Saturating add |
| QSUB | Saturating subtract |
| QDADD | Saturating double and add |
| QDSUB | Saturating double and subtract |
| SSAT | Signed saturate |
| USAT | Unsigned saturate |
| SSAT16 | Signed saturate 16-bit |
| USAT16 | Unsigned saturate 16-bit |

## 10. Parallel Add / Subtract (SIMD)

| Mnemonic | Description |
|----------|-------------|
| SADD16/SASX/SSAX/SSUB16/SADD8/SSUB8 | Signed parallel |
| QADD16/QASX/QSAX/QSUB16/QADD8/QSUB8 | Saturating signed parallel |
| SHADD16/SHASX/SHSAX/SHSUB16/SHADD8/SHSUB8 | Halving signed parallel |
| UADD16/UASX/USAX/USUB16/UADD8/USUB8 | Unsigned parallel |
| UQADD16/UQASX/UQSAX/UQSUB16/UQADD8/UQSUB8 | Saturating unsigned parallel |
| UHADD16/UHASX/UHSAX/UHSUB16/UHADD8/UHSUB8 | Halving unsigned parallel |

## 11. Packing / Unpacking / Reversal

| Mnemonic | Description |
|----------|-------------|
| PKHBT | Pack halfword bottom-top |
| PKHTB | Pack halfword top-bottom |
| SXTB/SXTH/SXTB16 | Sign extend (byte, halfword, byte16) |
| SXTAB/SXTAH/SXTAB16 | Sign extend and add |
| UXTB/UXTH/UXTB16 | Unsigned extend |
| UXTAB/UXTAH/UXTAB16 | Unsigned extend and add |
| REV | Byte-reverse word |
| REV16 | Byte-reverse packed halfwords |
| REVSH | Byte-reverse signed halfword |
| RBIT | Reverse bits |
| SEL | Select bytes |
| CLZ | Count leading zeros |
| USAD8/USADA8 | Unsigned sum of absolute differences |

## 12. Signed Multiply (DSP)

| Mnemonic | Description |
|----------|-------------|
| SMLABB/BT/TB/TT | Signed multiply halfwords, accumulate |
| SMLAWB/WT | Signed multiply word by halfword, accumulate |
| SMULBB/BT/TB/TT | Signed multiply halfwords |
| SMULWB/WT | Signed multiply word by halfword |
| SMLAD/X | Signed multiply-add dual |
| SMLSD/X | Signed multiply-subtract dual |
| SMMUL/R | Signed most-significant-word multiply |
| SMMLA/R | Signed most-significant-word multiply-accumulate |
| SMMLS/R | Signed most-significant-word multiply-subtract |
| SMUAD/X | Signed dual multiply-add |
| SMUSD/X | Signed dual multiply-subtract |
| SMLALBB/BT/TB/TT | Signed multiply halfwords, accumulate long |
| SMLALD/X | Signed multiply-add dual long |
| SMLSLD/X | Signed multiply-subtract dual long |

## 13. Divide

| Mnemonic | Description |
|----------|-------------|
| SDIV | Signed divide |
| UDIV | Unsigned divide |

## 14. Bit Field

| Mnemonic | Description |
|----------|-------------|
| BFC | Bit field clear |
| BFI | Bit field insert |
| SBFX | Signed bit field extract |
| UBFX | Unsigned bit field extract |

## 15. Exception / System

| Mnemonic | Description |
|----------|-------------|
| SVC | Supervisor call |
| SMC | Secure monitor call |
| HVC | Hypervisor call |
| UDF | Permanently undefined |
| ERET | Exception return |
| RFE | Return from exception |
| SRS | Store return state |
| SETEND | Set endianness |

## 16. Thumb-Specific Encodings

| Mnemonic | Description |
|----------|-------------|
| MOVS (T1) | Move with flags (Thumb short) |
| ADDS (T1) | Add with flags (Thumb short) |
| SUBS (T1) | Subtract with flags (Thumb short) |
| ADR | Form PC-relative address |
| ADDW | Add wide immediate (Thumb-2) |
| SUBW | Subtract wide immediate (Thumb-2) |

---

## Summary by Category

| #  | Category                    | Count |
|----|-----------------------------|-------|
| 1  | Data Processing             | 23    |
| 2  | Multiply                    | 8     |
| 3  | Load / Store (Single)       | 20    |
| 4  | Load / Store Multiple       | 4     |
| 5  | Branch                      | 10    |
| 6  | Status Register Access      | 3     |
| 7  | VFP / NEON Coprocessor      | 56+   |
| 8  | Miscellaneous / Hints       | 12    |
| 9  | Saturating Arithmetic       | 8     |
| 10 | Parallel Add/Sub (SIMD)     | 36    |
| 11 | Packing/Unpacking/Reversal  | 18+   |
| 12 | Signed Multiply (DSP)       | 30    |
| 13 | Divide                      | 2     |
| 14 | Bit Field                   | 4     |
| 15 | Exception / System          | 8     |
| 16 | Thumb-Specific              | 6     |
|    | **TOTAL**                   |**~250**|

---

## Notes

- **ARM/Thumb auto-detection** from IDA's `T` segment register.
- **IT block support** with full If-Then state machine for Thumb-2.
- **Barrel shifter** with all 5 shift types (LSL, LSR, ASR, ROR, RRX).
- **Condition codes**: all 15 ARM conditions supported.
- **Exception vectors** at 0x00-0x1C (Reset, Undef, SVC, Prefetch Abort, Data Abort, Hyp, IRQ, FIQ).
- **Big-endian support**: detected from IDA database configuration.
- **Call-depth tracking** with shadow LR stack for BL/BLX subroutine calls.
