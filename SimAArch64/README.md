# AArch64 Simulator -- IDA Pro Plugin (Prototype) by Luigi Origa

> **[!] This is a prototype / work-in-progress.**
> Features marked as *TODO* are planned but not available in the current build.

A lightweight **ARMv8-A (AArch64 / ARM64)** CPU simulator embedded inside **IDA Pro**.
It supports the full AArch64 instruction set including SIMD/NEON, floating-point
(single, double, half-precision), LSE atomics, CRC32, SHA/AES cryptography extensions,
and dot-product instructions.
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
  - [Opcode Categories 1-24](#1-data-processing----immediate)
  - [Summary](#summary-by-category)

---

# Part 1 -- Command Reference

## Keyboard Shortcuts

| Shortcut | Action |
|----------|--------|
| `Alt-3`  | Run the simulator on the current function |
| `Ctrl-3` | Show the command reference help |

---

## How It Works

Commands are placed as a **repeatable comment** (`;` in IDA's assembly view) at the very beginning of the function you want to simulate. The plugin parses these comments before execution begins.

```asm
; @x0 = 0x10
; @x1 = rand(0, 0xFF)
; @sp = stack(0)
; @max 5000
; @run
sub_FFFF0000:
    ...
```

The simulator reads the comment block, sets up the CPU state accordingly, and begins emulation. Output (register dumps, memory dumps, trace files) is written to the IDA output window and/or to disk depending on the save flags used.

The simulator supports all 4 exception levels (EL0-EL3) with VBAR-based vector dispatch.

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
@x0..@x30  = value|gen     ; Set 64-bit general-purpose register X0--X30
@w0..@w30  = value|gen     ; Set 32-bit view (zero-extends to 64)
@sp        = value|gen     ; Set Stack Pointer
@lr        = value|gen     ; Alias for X30 (Link Register)
@pc        = value         ; Set Program Counter (plain value only)
@pstate    = value|gen     ; Set Process State (NZCV + control bits)
@fpcr      = value|gen     ; Set Floating-Point Control Register
@fpsr      = value|gen     ; Set Floating-Point Status Register
@b0..@b31  = value|gen     ; Set SIMD 8-bit register
@h0..@h31  = value|gen     ; Set SIMD 16-bit register (half-precision)
@s0..@s31  = value|gen     ; Set SIMD 32-bit register (single)
@d0..@d31  = value|gen     ; Set SIMD 64-bit register (double)
@q0..@q31  = value|gen     ; Set SIMD 128-bit register (quadword)
@v0..@v31  = value|gen     ; Set full 128-bit vector register
```

**System registers (assignable):**

```
@spsr_el1  = value|gen     ; Saved Program Status Register (EL1)
@elr_el1   = value|gen     ; Exception Link Register (EL1)
@vbar_el1  = value|gen     ; Vector Base Address Register (EL1)
@tpidr_el0 = value|gen     ; Thread Pointer (EL0)
@sctlr_el1 = value|gen     ; System Control Register (EL1)
```

**Examples:**

```
; @x0  = 0
; @x1  = rand(0, 0xFFFF)
; @sp  = 0x80000000
; @pstate = flag(n=0, z=0, c=0, v=0)
; @pc  = 0xFFFF000000001000
```

---

## Simulation Commands

| Command | Description |
|---------|-------------|
| `@run` | Run or resume the simulator |
| `@max <N>` | Maximum number of instructions to execute |
| `@timeout <ms>` | Simulation timeout in milliseconds |
| `@loop_limit <N>` | Infinite-loop detection check interval (default 1000) |
| `@debug` | Enable per-function debug log |

Multiple `@run` commands in sequence resume emulation from the current PC.

---

## Breakpoints

| Command | Description |
|---------|-------------|
| `@bp <addr>` | Breakpoint at the given PC address |
| `@bp Xn op value` | Break when 64-bit register `Xn` satisfies condition |
| `@bp Wn op value` | Break when 32-bit register `Wn` satisfies condition |
| `@bp memN[addr] op val` | Break on memory condition (`N` = 8, 16, 32, 64) |

Supported operators: `==`  `!=`  `>`  `<`  `>=`  `<=`

Breakpoints accumulate across multiple `@run` invocations.

**Examples:**

```
; @bp 0xFFFF000000003FFC
; @bp X5 >= 0x100
; @bp mem32[0x80002000] != 0
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
; @mem32 [0x80001000] = 0xDEADBEEF
; @mem8  [0x80002000] = list(0x01, 0x02, 0x03, 0xFF)
; @mem16 [0x80003000-0x80003020] = 0x0000
; @mem64 [0x80004000] = rand(0, 0xFFFFFFFFFFFFFFFF)
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
; @dump64(0x80001000, 128)
; @save_memory("C:/traces/heap_after.bin", 0x80001000, 0x200)
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
; @mock_read64 0xFFFF000010000000 = 0xCAFEBABEDEADBEEF
; @mock_read8  0xFFFF000020000000 = list(0x00, 0x01, 0x02), loop
; @mock_read32 0xFFFF000030000000 = rand(0, 0xFFFFFFFF)
; @mock_read64 0xFFFF000040000000 = list(10, 20, 30), loop(1, 5)
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

### Group 2 -- Register-relative *(registers, memory, fuzz)*

| Generator | Description |
|-----------|-------------|
| `neg(Xn\|Wn)` | Arithmetic negation of the register value |
| `mirror(Xn\|Wn)` | Bit-reversal of the register value |
| `arith(Xn\|Wn, op, val)` | Arithmetic on register: `+` `-` `*` `/` `&` `\|` `^` `~` `%` `L`(shl) `R`(shr) `A`(asr) `a`(abs) `z`(clz) |
| `off(Xn, val)` | Register value plus a signed offset |
| `aligned(Xn, n)` | Register aligned down to `n` bytes |
| `overflow(Xn)` | Register + MAX_SIGNED |
| `underflow(Xn)` | Register - MIN_SIGNED |

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
| `stack(offset)` | Stack pointer (`SP`) plus a signed offset |

### Group 4 -- Iteration-based *(fuzz only)*

| Generator | Description |
|-----------|-------------|
| `range(min, max [,step])` | Iterate from `min` to `max` by `step` (default 1) |
| `edges` | Boundary values: 0, 1, 0x7F, 0x80, 0xFF, 0x7FFF, ..., MAX_64 |
| `signed_edges` | Signed boundaries for 64-bit |
| `pow2` | Powers of 2: 1, 2, 4, ..., 2^63 |
| `pow2_minus1` | Powers of 2 minus 1: 0, 1, 3, 7, ..., 0xFFFFFFFFFFFFFFFF |
| `rot_walk` | Rotating single-bit walk |
| `byte_walk` | Walking byte: 0xFF across all 8 byte positions |
| `walk(val)` | XOR base with walking bit: val ^ (1 << (it%64)) |
| `flip(addr, n)` | Bit-flip: addr ^ (1 << (it%n)) |

### Special

| Generator | Description |
|-----------|-------------|
| `flag(n=v, z=v, c=v, v=v)` | Compose a PSTATE value from individual flag assignments |

**Flag names:** `N`, `Z`, `C`, `V`, `D`, `A`, `I`, `F`

**Example:**

```
; @pstate = flag(n=0, z=0, c=0, v=0)
```

---

## Fuzz Block

The fuzz block lets you run the function repeatedly while varying one or more inputs across a defined space. Each fuzz step re-initializes CPU state from the register assignments above, applies the fuzz target value, and runs the simulation.

```
@fuzz
  seed(N)
  sequence:
  parallel=N
  resume(file)
  <iterations> [call(addr)] TARGET = gen [stop=cond] [log=type] [options...]
```

### Targets

| Target | Description |
|--------|-------------|
| `X0`..`X30` | 64-bit general-purpose register |
| `W0`..`W30` | 32-bit general-purpose register |
| `SP` | Stack pointer |
| `B0`..`B31`, `H0`..`H31`, `S0`..`S31`, `D0`..`D31`, `Q0`..`Q31`, `V0`..`V31` | SIMD/FP registers |
| `SP[offset]` | Stack memory at `SP + offset` |
| `mem8[addr]` .. `mem64[addr]` | Memory location (8/16/32/64-bit) |

### Stop Conditions

| Condition | Description |
|-----------|-------------|
| `crash` | Stop on any crash/fault |
| `any_exception` | Stop on any exception |
| `pc(addr)` | Stop when PC reaches address |
| `mem_write(addr)` | Stop on write to address |
| `reg(Xn, val)` | Stop when register equals value |
| `Xn==val` | Stop when register equals value |
| `Xn!=val` | Stop when register differs |

### Log Types

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
; @x0  = 0x80001000
; @x1  = 0x80001000
; @max 2000
; @fuzz
;   0 X1 = range(0, 0xFF, 1)
;   1 X1 = rand(0, 0xFFFFFFFFFFFFFFFF)
;   2 mem64[0x80001000] = rand(0, 0xFFFFFFFFFFFFFFFF)
; @save_fuzz_results
```

---

## Notes

- This is a **prototype**: stability, accuracy, and feature completeness are not guaranteed.
- 64-bit address space with 16 MiB flat simulated memory buffer.
- Full EL0-EL3 exception model with VBAR-based vector dispatch.
- Auto SP init (if SP==0, auto-initialized to middle of memory buffer, 16-byte aligned).
- Auto LR init (X30 set to end_addr so RET naturally terminates simulation).
- Background thread execution with UI progress bar and 30-second timeout.
- Batch mode (`-A` flag) for headless/idalib operation.
- Feedback and bug reports are welcome via the issue tracker.

---

# Part 2 -- Supported Opcodes

## 1. Data Processing -- Immediate

| Mnemonic | Description |
|----------|-------------|
| ADD/ADDS | Add (32/64-bit immediate) |
| SUB/SUBS | Subtract (32/64-bit immediate) |
| AND/ANDS | Bitwise AND (bitmask immediate) |
| ORR | Bitwise OR (bitmask immediate) |
| EOR | Bitwise XOR (bitmask immediate) |
| MOVN | Move wide NOT |
| MOVZ | Move wide zero |
| MOVK | Move wide keep |
| ADR | PC-relative address |
| ADRP | PC-relative address (page) |

## 2. Data Processing -- Register

| Mnemonic | Description |
|----------|-------------|
| ADD/ADDS | Add (shifted/extended register) |
| SUB/SUBS | Subtract (shifted/extended register) |
| ADC/ADCS | Add with carry |
| SBC/SBCS | Subtract with carry |
| AND/ANDS | Bitwise AND (shifted register) |
| ORR | Bitwise OR (shifted register) |
| ORN | Bitwise OR NOT |
| EOR | Bitwise XOR (shifted register) |
| EON | Bitwise XOR NOT |
| BIC/BICS | Bit clear (shifted register) |

## 3. Shift / Bitfield / Extract

| Mnemonic | Description |
|----------|-------------|
| LSLV | Logical shift left variable |
| LSRV | Logical shift right variable |
| ASRV | Arithmetic shift right variable |
| RORV | Rotate right variable |
| BFM | Bitfield move |
| SBFM | Signed bitfield move (SXTB/SXTH/SXTW/ASR/SBFX/SBFIZ) |
| UBFM | Unsigned bitfield move (UXTB/UXTH/LSL/LSR/UBFX/UBFIZ) |
| EXTR | Extract (covers ROR immediate) |

## 4. Multiply / Divide

| Mnemonic | Description |
|----------|-------------|
| MADD | Multiply-add (covers MUL alias) |
| MSUB | Multiply-subtract (covers MNEG alias) |
| SMADDL | Signed multiply-add long |
| SMSUBL | Signed multiply-subtract long |
| UMADDL | Unsigned multiply-add long |
| UMSUBL | Unsigned multiply-subtract long |
| SMULH | Signed multiply high |
| UMULH | Unsigned multiply high |
| SDIV | Signed divide |
| UDIV | Unsigned divide |

## 5. Load / Store (Single Register)

| Mnemonic | Description |
|----------|-------------|
| LDR | Load (32/64-bit, all addressing modes) |
| STR | Store (32/64-bit, all addressing modes) |
| LDRB | Load byte |
| LDRH | Load halfword |
| LDRSB | Load signed byte |
| LDRSH | Load signed halfword |
| LDRSW | Load signed word |
| STRB | Store byte |
| STRH | Store halfword |
| PRFM | Prefetch memory |

## 6. Load / Store Pair

| Mnemonic | Description |
|----------|-------------|
| LDP | Load pair (32/64-bit) |
| STP | Store pair (32/64-bit) |
| LDPSW | Load pair signed word |
| LDP/STP SIMD | Load/store pair (32/64/128-bit SIMD) |

## 7. Load / Store Exclusive / Ordered

| Mnemonic | Description |
|----------|-------------|
| LDXR/STXR | Load/store exclusive (32/64-bit) |
| LDXP/STXP | Load/store exclusive pair |
| LDAXR/STLXR | Load-acquire exclusive / store-release exclusive |
| LDAR/STLR | Load-acquire / store-release |
| CAS | Compare-and-swap (32/64-bit) |

## 8. Branch

| Mnemonic | Description |
|----------|-------------|
| B | Branch (unconditional) |
| BL | Branch with link |
| BR | Branch to register |
| BLR | Branch with link to register |
| RET | Return |
| B.cond | Branch conditional (EQ/NE/CS/CC/MI/PL/VS/VC/HI/LS/GE/LT/GT/LE/AL/NV) |
| CBZ/CBNZ | Compare and branch if zero/non-zero |
| TBZ/TBNZ | Test bit and branch if zero/non-zero |

## 9. Conditional Select

| Mnemonic | Description |
|----------|-------------|
| CSEL | Conditional select |
| CSINC | Conditional select increment (covers CSET, CINC) |
| CSINV | Conditional select invert (covers CSETM, CINV) |
| CSNEG | Conditional select negate (covers CNEG) |

## 10. Conditional Compare

| Mnemonic | Description |
|----------|-------------|
| CCMN | Conditional compare negative (register/immediate) |
| CCMP | Conditional compare (register/immediate) |

## 11. System / Hints

| Mnemonic | Description |
|----------|-------------|
| NOP | No operation |
| WFI | Wait for interrupt |
| WFE | Wait for event |
| SEV/SEVL | Send event / send event local |
| YIELD | Yield |
| ISB | Instruction synchronization barrier |
| DSB | Data synchronization barrier |
| DMB | Data memory barrier |
| CLREX | Clear exclusive |
| MSR | Move to system register |
| MRS | Move from system register |

## 12. Exception Generation

| Mnemonic | Description |
|----------|-------------|
| SVC | Supervisor call |
| HVC | Hypervisor call |
| SMC | Secure monitor call |
| BRK | Breakpoint |
| HLT | Halt |
| DCPS1/2/3 | Debug change PE state |
| ERET | Exception return |
| DRPS | Debug restore PE state |

## 13. Floating-Point Scalar

| Mnemonic | Description |
|----------|-------------|
| FADD/FSUB/FMUL/FDIV | FP arithmetic (single/double) |
| FNEG/FABS/FSQRT | FP unary (single/double) |
| FMADD/FMSUB/FNMADD/FNMSUB | Fused multiply-add (single/double) |
| FCMP | FP compare (single/double) |
| FCCMP | FP conditional compare |
| FCSEL | FP conditional select |
| FMOV | FP move (register, GP<->FP, immediate) |
| FMIN/FMAX/FMINNM/FMAXNM | FP min/max (single/double) |

## 14. Floating-Point Conversion

| Mnemonic | Description |
|----------|-------------|
| FCVTZS/FCVTZU | FP to integer (all width combinations) |
| SCVTF/UCVTF | Integer to FP (all width combinations) |
| FCVT | Precision conversion (S<->D) |
| FRINTN/FRINTP/FRINTM/FRINTZ | FP rounding (single/double) |

## 15. SIMD (Advanced SIMD / NEON)

| Mnemonic | Description |
|----------|-------------|
| ADD/SUB | Vector integer add/subtract |
| AND/ORR/EOR/BIC/ORN | Vector logical |
| BSL/BIT/BIF/NOT | Vector bitwise select/insert |
| MUL/MLA/MLS | Vector integer multiply |
| SHL/SSHR/USHR | Vector shift |
| CMEQ/CMGT/CMGE/CMHI/CMHS | Vector compare |
| FADD/FSUB/FMUL | Vector FP arithmetic |
| DUP | Duplicate (from GP or element) |
| INS | Insert (from GP or element) |
| UMOV/SMOV | Move from vector element |
| MOV | Vector move |
| LDR/STR SIMD | Load/store SIMD (8/16/32/64/128-bit) |

## 16. Bit Manipulation

| Mnemonic | Description |
|----------|-------------|
| CLZ | Count leading zeros (32/64-bit) |
| CLS | Count leading sign bits |
| RBIT | Reverse bits (32/64-bit) |
| REV | Reverse bytes in word/doubleword |
| REV16 | Reverse bytes in halfwords |
| REV32 | Reverse bytes in words |

## 17. SIMD Advanced

| Mnemonic | Description |
|----------|-------------|
| ADDP/FADDP | Pairwise add |
| ADDV/SMAXV/SMINV/UMAXV/UMINV | Across-vector reduction |
| SMAX/SMIN/UMAX/UMIN | Vector min/max |
| SADDL/UADDL/SSUBL/USUBL | Widening add/subtract |
| SADDW/UADDW | Wide add |
| XTN/SQXTN/UQXTN/SQXTUN | Narrowing |
| SXTL/UXTL | Extending (long) |
| SABD/UABD/SABA/UABA | Absolute difference |
| ZIP1/ZIP2/UZP1/UZP2/TRN1/TRN2 | Permutation |
| EXT | Extract |
| TBL/TBX | Table lookup |
| NEG/ABS/FNEG/FABS | Vector negate/absolute |
| FMAX/FMIN | Vector FP min/max |
| FCMEQ/FCMGT/FCMGE | Vector FP compare |
| FCVTZS/FCVTZU/SCVTF/UCVTF | Vector FP conversion |
| SMULL/UMULL | Widening multiply |
| SQADD/UQADD/SQSUB/UQSUB | Saturating arithmetic |
| SLI/SRI | Shift and insert |
| SMLAL/UMLAL | Multiply-accumulate long |
| MOVI | Move immediate (vector) |

## 18. LSE Atomics (ARMv8.1)

| Mnemonic | Description |
|----------|-------------|
| LDADD/LDADDA/LDADDL/LDADDAL | Atomic add (32/64) |
| LDCLR/LDSET/LDEOR | Atomic clear/set/XOR (32/64) |
| SWP/SWPA/SWPL/SWPAL | Atomic swap (32/64) |
| STADD/STCLR/STSET/STEOR | Atomic store operations (32/64) |
| CASP | Compare-and-swap pair (32/64) |

## 19. CRC32

| Mnemonic | Description |
|----------|-------------|
| CRC32B/CRC32H/CRC32W/CRC32X | CRC-32 (byte/half/word/doubleword) |
| CRC32CB/CRC32CH/CRC32CW/CRC32CX | CRC-32C (byte/half/word/doubleword) |

## 20. Half-Precision Floating-Point (ARMv8.2-FP16)

| Mnemonic | Description |
|----------|-------------|
| FADD_H/FSUB_H/FMUL_H/FDIV_H | Half-precision arithmetic |
| FNEG_H/FABS_H/FSQRT_H | Half-precision unary |
| FMADD_H/FMSUB_H/FNMADD_H/FNMSUB_H | Half-precision fused multiply-add |
| FCMP_H/FCSEL_H/FMOV_H | Half-precision compare/select/move |
| FCVT (half<->single, half<->double) | Half-precision conversion |
| FCVTZS/FCVTZU from half | Half-precision to integer |
| SCVTF/UCVTF to half | Integer to half-precision |

## 21. Flag Manipulation (ARMv8.4/8.5)

| Mnemonic | Description |
|----------|-------------|
| CFINV | Invert carry flag |
| SETF8 | Set flags from 8-bit value |
| SETF16 | Set flags from 16-bit value |
| RMIF | Rotate mask insert flags |

## 22. SIMD By-Element Operations

| Mnemonic | Description |
|----------|-------------|
| MUL/MLA/MLS (by element) | Integer multiply by element |
| SMULL/UMULL/SMLAL/UMLAL (by element) | Widening multiply by element |
| FMUL/FMLA/FMLS (by element) | FP multiply by element |

## 23. Dot Product (ARMv8.2-DotProd)

| Mnemonic | Description |
|----------|-------------|
| SDOT | Signed dot product (vector and by-element) |
| UDOT | Unsigned dot product (vector and by-element) |

## 24. Cryptography Extensions (ARMv8.0)

| Mnemonic | Description |
|----------|-------------|
| AESE/AESD/AESMC/AESIMC | AES encrypt/decrypt/mix/inv-mix columns |
| SHA1C/SHA1P/SHA1M/SHA1H/SHA1SU0/SHA1SU1 | SHA-1 operations |
| SHA256H/SHA256H2/SHA256SU0/SHA256SU1 | SHA-256 operations |

---

## Summary by Category

| #  | Category                       | Count |
|----|--------------------------------|-------|
| 1  | Data Processing (Immediate)    | 10    |
| 2  | Data Processing (Register)     | 10    |
| 3  | Shift / Bitfield / Extract     | 8     |
| 4  | Multiply / Divide              | 10    |
| 5  | Load / Store (Single)          | 10    |
| 6  | Load / Store Pair              | 4     |
| 7  | Load / Store Exclusive/Ordered | 9     |
| 8  | Branch                         | 8     |
| 9  | Conditional Select             | 4     |
| 10 | Conditional Compare            | 2     |
| 11 | System / Hints                 | 11    |
| 12 | Exception Generation           | 8     |
| 13 | Floating-Point Scalar          | 20+   |
| 14 | Floating-Point Conversion      | 10+   |
| 15 | SIMD (NEON) Basic              | 30+   |
| 16 | Bit Manipulation               | 6     |
| 17 | SIMD Advanced                  | 50+   |
| 18 | LSE Atomics                    | 18    |
| 19 | CRC32                          | 8     |
| 20 | Half-Precision FP              | 20+   |
| 21 | Flag Manipulation              | 4     |
| 22 | SIMD By-Element                | 12+   |
| 23 | Dot Product                    | 2     |
| 24 | Cryptography (AES/SHA)         | 14    |
|    | **TOTAL**                      |**~300+**|

---

## Notes

- **Full AArch64 ISA** with ARMv8.0 base, ARMv8.1 LSE atomics, ARMv8.2 FP16 + DotProd, ARMv8.4/8.5 flag manipulation.
- **Exception levels EL0-EL3** with VBAR-based vector dispatch, SPSR/ELR/ESR save.
- **SIMD/NEON**: full 128-bit vector register file (V0-V31) with all scalar views (B/H/S/D/Q).
- **Cryptography**: AES and SHA-1/SHA-256 hardware acceleration instructions.
- **16 condition codes** supported for B.cond, CSEL, CCMP, FCCMP, FCSEL.
