# TriCore Simulator -- IDA Pro Plugin (Prototype)

> **[!] This is a prototype / work-in-progress.**
> Features marked as *TODO* are planned but not available in the current build.

A lightweight **Infineon TriCore** CPU simulator embedded inside **IDA Pro**.
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
  - [CPU Core Selection](#cpu-core-selection)
  - [Memory Write](#memory-write)
  - [Memory Dump](#memory-dump)
  - [Mock Reads](#mock-reads)
  - [Save Flags](#save-flags)
  - [Generators](#generators)
  - [Fuzz Block](#fuzz-block)
  - [Notes](#notes)
- [Part 2 -- Supported Opcodes](#part-2----supported-opcodes)
  - [Opcode Categories 1-27](#1-system--special-instructions)
  - [Summary](#summary-by-category)
  - [Missing / Unsupported](#missing--unsupported-opcodes)

---

# Part 1 -- Command Reference

## Keyboard Shortcuts

| Shortcut | Action |
|----------|--------|
| `Alt-0`  | Run the simulator on the current function |
| `Ctrl-0` | Show the command reference help |

---

## How It Works

Commands are placed as a **repeatable comment** (`;` in IDA's assembly view) at the very beginning of the function you want to simulate. The plugin parses these comments before execution begins.

```asm
; @d0 = 0x10
; @d1 = rand(0, 0xFF)
; @a4 = stack(0)
; @max 5000
; @run
sub_8001234:
    ...
```

The simulator reads the comment block, sets up the CPU state accordingly, and begins emulation. Output (register dumps, memory dumps, trace files) is written to the IDA output window and/or to disk depending on the save flags used.

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
@d0..@d15  = value|gen     ; Set data register D0--D15
@a0..@a15  = value|gen     ; Set address register A0--A15
@psw       = value|gen     ; Set PSW (Program Status Word)
@pc        = value         ; Set Program Counter (plain value only)
@pcxi      = value|gen     ; Set PCXI register
@fcx       = value|gen     ; Set FCX register
@lcx       = value|gen     ; Set LCX register
@isp       = value|gen     ; Set Interrupt Stack Pointer
@icr       = value|gen     ; Set Interrupt Control Register
@biv       = value|gen     ; Set Base Address of Interrupt Vector Table
@btv       = value|gen     ; Set Base Address of Trap Vector Table
```

**Examples:**

```
; @d0  = 0
; @d1  = rand(0, 0xFFFF)
; @a4  = 0xD0001000
; @psw = flag(C=0, V=0)
; @pc  = 0x80012000
```

---

## Simulation Commands

| Command | Description |
|---------|-------------|
| `@run` | Run or resume the simulator |
| `@max <N>` | Maximum number of instructions to execute (0 = unlimited) |
| `@timeout <ms>` | Simulation timeout in milliseconds |
| `@loop_limit <N>` | Infinite-loop detection check interval (default: 1000) |
| `@debug` | Enable per-function debug log |

Multiple `@run` commands in sequence resume emulation from the current PC.

---

## Breakpoints

| Command | Description |
|---------|-------------|
| `@bp <addr>` | Breakpoint at the given PC address |
| `@bp Dn op value` | Break when data register `Dn` satisfies condition |
| `@bp An op value` | Break when address register `An` satisfies condition |
| `@bp memN[addr] op val` | Break on memory condition (`N` = 8, 16, 32, 64) |

Supported operators: `==`  `!=`  `>`  `<`  `>=`  `<=`

Breakpoints accumulate across multiple `@run` invocations.

**Examples:**

```
; @bp 0x80013FFC
; @bp D5 >= 0x100
; @bp mem32[0xD0002000] != 0
; @max 10000
; @run
```

---

## CPU Core Selection

Select which TriCore CPU core to target for simulation.

```
@cpu0 .. @cpu6
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
; @mem32 [0xD0001000] = 0xDEADBEEF
; @mem8  [0xD0002000] = list(0x01, 0x02, 0x03, 0xFF)
; @mem16 [0xD0003000-0xD0003020] = 0x0000
; @mem32 [0xD0004000] = rand(0, 0xFFFFFFFF)
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
; @dump32(0xD0001000, 64)
; @save_memory("C:/traces/heap_after.bin", 0xD0001000, 0x200)
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
; @mock_read32 0xF0000010 = 0xCAFEBABE
; @mock_read8  0xF0000020 = list(0x00, 0x01, 0x02), loop
; @mock_read16 0xF0000030 = rand(0, 0xFFFF)
; @mock_read32 0xF0000040 = list(10, 20, 30), loop(1, 5)
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

### Group 2 -- Register-relative *(D/A registers, memory, fuzz)*

| Generator | Description |
|-----------|-------------|
| `neg(Dn\|An)` | Arithmetic negation of the register value |
| `mirror(Dn\|An)` | Bit-reversal of the register value |
| `arith(Dn\|An, op, val)` | Arithmetic on register: `+` `-` `*` `/` `&` `\|` `^` |
| `off(An, val)` | Address register value plus a signed offset |
| `aligned(An, n)` | Address register aligned down to `n` bytes |
| `overflow(An)` | Address register + 0x7FFFFFFF |
| `underflow(An)` | Address register - 0x7FFFFFFF |

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

### Group 3 -- Pointer *(A registers, memory, fuzz)*

| Generator | Description |
|-----------|-------------|
| `stack(offset)` | Stack pointer (`A10`) plus a signed offset |

### Group 4 -- Iteration-based *(fuzz only)*

| Generator | Description |
|-----------|-------------|
| `range(min, max [,step])` | Iterate from `min` to `max` by `step` (default 1) |
| `edges` | Boundary values: 0, 1, 0x7FFFFFFF, 0x80000000, 0xFFFFFFFE, 0xFFFFFFFF |
| `signed_edges` | Signed boundaries: -1, -128, -32768, -2^31, 0, 127, 32767 |
| `pow2` | Powers of 2: 1, 2, 4, ..., 2^31 |
| `pow2_minus1` | Powers of 2 minus 1: 0, 1, 3, 7, ..., 0xFFFFFFFF |
| `rot_walk` | Rotating single-bit walk (same as pow2) |
| `byte_walk` | Walking byte: 0xFF, 0xFF00, 0xFF0000, 0xFF000000 |
| `walk(val)` | XOR base with walking bit: val ^ (1 << (it%32)) |
| `flip(addr, n)` | Bit-flip: addr ^ (1 << (it%n)) |

### Special

| Generator | Description |
|-----------|-------------|
| `flag(C=v, V=v, ...)` | Compose a PSW value from individual flag assignments |

**Flag names:** `C`, `V`, `SV`, `AV`, `SAV`, `IS`, `GW`, `CDE`, `CDC`, `FPU_RM`, `S`, `IO`, `PRS`, `ID`, `UM`, `PRIV`

**Example:**

```
; @psw = flag(C=0, V=0, SV=0, AV=0)
```

---

## Fuzz Block

The fuzz block lets you run the function repeatedly while varying one or more inputs across a defined space. Each fuzz step re-initializes CPU state from the register assignments above, applies the fuzz target value, and runs the simulation.

```
@fuzz
  <step_index> [call(addr)] TARGET = gen [stop=cond] [options...]
```

### Targets

| Target | Description |
|--------|-------------|
| `D0`..`D15` | Data register |
| `A0`..`A15` | Address register |
| `PSW`, `FCX`, `PCXI`, `ICR` | Special-purpose registers |
| `SP[offset]` | Stack memory at `SP + offset` |
| `mem8[addr]` .. `mem64[addr]` | Memory location (8/16/32/64-bit) |

**Example:**

```
; @d0  = 0xD0001000
; @a4  = 0xD0001000
; @max 2000
; @fuzz
;   0 D1 = range(0, 0xFF, 1)
;   1 D1 = rand(0, 0xFFFFFFFF)
;   2 mem32[0xD0001000] = rand(0, 0xFFFFFFFF)
; @save_fuzz_results
```

- The `step_index` (0, 1, 2, ...) controls the order of fuzz steps within one iteration.
- `call(addr)` optionally redirects execution to a different entry point for that step.
- Each step's generator is evaluated fresh for every iteration.
- `@save_fuzz_results` collects register state, PC, and any trap information for each iteration.

---

## Notes

- This is a **prototype**: stability, accuracy, and feature completeness are not guaranteed.
- Feedback and bug reports are welcome via the issue tracker.

---

# Part 2 -- Supported Opcodes

## 1. System / Special Instructions

| Mnemonic | Description                     |
|----------|---------------------------------|
| NOP      | No operation                    |
| DEBUG    | Debug breakpoint                |
| RET      | Return from subroutine          |
| RFE      | Return from exception           |
| RFH      | Return from handler             |
| RFM      | Return from monitor             |
| FRET     | Return from fast call           |
| SVLCX    | Save lower context              |
| RSLCX    | Restore lower context           |
| STLCX    | Store lower context             |
| STUCX    | Store upper context             |
| LDLCX    | Load lower context              |
| LDUCX    | Load upper context              |
| ENABLE   | Enable interrupts               |
| DISABLE  | Disable interrupts              |
| RESTORE  | Restore state                   |
| WAIT     | Wait for interrupt              |
| ISYNC    | Instruction synchronize         |
| DSYNC    | Data synchronize                |
| LSYNC    | Load synchronize                |
| TRAPV    | Trap on overflow                |
| TRAPSV   | Trap on sticky overflow         |
| TRAPINV  | Trap on invalid                 |
| RSTV     | Reset overflow bits             |
| SYSCALL  | System call                     |
| HVCALL   | Hypervisor call                 |
| BISR     | Begin interrupt service routine |
| UPDFL    | Update FPU flags                |

## 2. Move / Load Immediate

| Mnemonic | Description                   |
|----------|-------------------------------|
| MOV      | Move (register/immediate)     |
| MOV.A    | Move to address register      |
| MOV.AA   | Move address to address       |
| MOV.D    | Move data register to address |
| MOV.U    | Move unsigned immediate       |
| MOVH     | Move high immediate           |
| MOVH.A   | Move high to address register |
| CMOV     | Conditional move              |
| CMOVN    | Conditional move (negated)    |

## 3. Address Register Operations

| Mnemonic | Description                   |
|----------|-------------------------------|
| ADD.A    | Add address registers         |
| SUB.A    | Subtract address registers    |
| ADDSC.A  | Add scaled address            |
| ADDSC.AT | Add scaled address (tagged)   |
| ADDIH.A  | Add immediate high to address |
| EQ.A     | Equal address                 |
| NE.A     | Not equal address             |
| LT.A     | Less than address             |
| GE.A     | Greater/equal address         |
| EQZ.A    | Equal zero address            |
| NEZ.A    | Not zero address              |

## 4. Integer Arithmetic -- Word

| Mnemonic | Description                         |
|----------|-------------------------------------|
| ADD      | Add                                 |
| ADDI     | Add immediate                       |
| ADDIH    | Add immediate high                  |
| ADDS     | Add saturated                       |
| ADDS.U   | Add saturated unsigned              |
| ADDX     | Add extended                        |
| ADDC     | Add with carry                      |
| SUB      | Subtract                            |
| SUBS     | Subtract saturated                  |
| SUBS.U   | Subtract saturated unsigned         |
| SUBX     | Subtract extended                   |
| SUBC     | Subtract with carry                 |
| RSUB     | Reverse subtract                    |
| RSUBS    | Reverse subtract saturated          |
| RSUBS.U  | Reverse subtract saturated unsigned |
| ABSDIF   | Absolute difference                 |
| ABSDIFS  | Absolute difference saturated       |
| ABS      | Absolute value                      |
| ABSS     | Absolute saturated                  |
| MIN      | Minimum                             |
| MIN.U    | Minimum unsigned                    |
| MAX      | Maximum                             |
| MAX.U    | Maximum unsigned                    |
| NOT      | Bitwise NOT                         |

## 5. Integer Arithmetic -- Byte

| Mnemonic | Description                      |
|----------|----------------------------------|
| ADD.B    | Add packed bytes                 |
| SUB.B    | Subtract packed bytes            |
| ABSDIF.B | Absolute difference packed bytes |
| ABS.B    | Absolute packed bytes            |
| EQ.B     | Equal packed bytes               |
| LT.B     | Less than packed bytes           |
| LT.BU    | Less than unsigned packed bytes  |
| EQANY.B  | Equal any packed bytes           |
| MIN.B    | Minimum packed bytes             |
| MIN.BU   | Minimum unsigned packed bytes    |
| MAX.B    | Maximum packed bytes             |
| MAX.BU   | Maximum unsigned packed bytes    |
| SAT.B    | Saturate to byte                 |
| SAT.BU   | Saturate to byte unsigned        |

## 6. Integer Arithmetic -- Halfword

| Mnemonic  | Description                                    |
|-----------|------------------------------------------------|
| ADD.H     | Add packed halfwords                           |
| ADDS.H    | Add saturated packed halfwords                 |
| ADDS.HU   | Add saturated unsigned packed halfwords        |
| SUB.H     | Subtract packed halfwords                      |
| SUBS.H    | Subtract saturated packed halfwords            |
| SUBS.HU   | Subtract saturated unsigned packed halfwords   |
| ABSDIF.H  | Absolute difference packed halfwords           |
| ABSDIFS.H | Absolute difference saturated packed halfwords |
| ABS.H     | Absolute packed halfwords                      |
| ABSS.H    | Absolute saturated packed halfwords            |
| EQ.H      | Equal packed halfwords                         |
| LT.H      | Less than packed halfwords                     |
| LT.HU     | Less than unsigned packed halfwords            |
| EQANY.H   | Equal any packed halfwords                     |
| MIN.H     | Minimum packed halfwords                       |
| MIN.HU    | Minimum unsigned packed halfwords              |
| MAX.H     | Maximum packed halfwords                       |
| MAX.HU    | Maximum unsigned packed halfwords              |
| SAT.H     | Saturate to halfword                           |
| SAT.HU    | Saturate to halfword unsigned                  |

## 7. Comparison / Conditional Arithmetic

| Mnemonic | Description                    |
|----------|--------------------------------|
| EQ       | Equal                          |
| NE       | Not equal                      |
| LT       | Less than                      |
| LT.U     | Less than unsigned             |
| GE       | Greater/equal                  |
| GE.U     | Greater/equal unsigned         |
| EQ.W     | Equal word                     |
| LT.W     | Less than word                 |
| LT.WU    | Less than word unsigned        |
| CADD     | Conditional add                |
| CADDN    | Conditional add (negated)      |
| CSUB     | Conditional subtract           |
| CSUBN    | Conditional subtract (negated) |
| SEL      | Conditional select             |
| SELN     | Conditional select (negated)   |

## 8. Conditional Bit Operations (AND/OR/XOR + comparison)

| Mnemonic | Description                       |
|----------|-----------------------------------|
| AND.EQ   | AND with equal                    |
| AND.NE   | AND with not-equal                |
| AND.LT   | AND with less-than                |
| AND.LT.U | AND with less-than unsigned       |
| AND.GE   | AND with greater/equal            |
| AND.GE.U | AND with greater/equal unsigned   |
| OR.EQ    | OR with equal                     |
| OR.NE    | OR with not-equal                 |
| OR.LT    | OR with less-than                 |
| OR.LT.U  | OR with less-than unsigned        |
| OR.GE    | OR with greater/equal             |
| OR.GE.U  | OR with greater/equal unsigned    |
| XOR.EQ   | XOR with equal                    |
| XOR.NE   | XOR with not-equal                |
| XOR.LT   | XOR with less-than                |
| XOR.LT.U | XOR with less-than unsigned       |
| XOR.GE   | XOR with greater/equal            |
| XOR.GE.U | XOR with greater/equal unsigned   |
| SH.EQ    | Shift with equal                  |
| SH.NE    | Shift with not-equal              |
| SH.LT    | Shift with less-than              |
| SH.LT.U  | Shift with less-than unsigned     |
| SH.GE    | Shift with greater/equal          |
| SH.GE.U  | Shift with greater/equal unsigned |

## 9. Logical Operations

| Mnemonic | Description     |
|----------|-----------------|
| AND      | Bitwise AND     |
| NAND     | Bitwise NAND    |
| OR       | Bitwise OR      |
| NOR      | Bitwise NOR     |
| XOR      | Bitwise XOR     |
| XNOR     | Bitwise XNOR    |
| ANDN     | Bitwise AND-NOT |
| ORN      | Bitwise OR-NOT  |

## 10. Shift / Rotate / Count

| Mnemonic | Description                        |
|----------|------------------------------------|
| SH       | Shift                              |
| SHA      | Shift arithmetic                   |
| SHAS     | Shift arithmetic saturated         |
| SH.H     | Shift packed halfwords             |
| SHA.H    | Shift arithmetic packed halfwords  |
| CLZ      | Count leading zeros                |
| CLO      | Count leading ones                 |
| CLS      | Count leading sign bits            |
| CLZ.H    | Count leading zeros (halfword)     |
| CLO.H    | Count leading ones (halfword)      |
| CLS.H    | Count leading sign bits (halfword) |
| SHUFFLE  | Shuffle bytes                      |

## 11. Bit Operations on Individual Bits

| Mnemonic   | Description        |
|------------|--------------------|
| AND.T      | AND bit            |
| NAND.T     | NAND bit           |
| OR.T       | OR bit             |
| NOR.T      | NOR bit            |
| XOR.T      | XOR bit            |
| XNOR.T     | XNOR bit           |
| ANDN.T     | AND-NOT bit        |
| ORN.T      | OR-NOT bit         |
| SH.AND.T   | Shift AND bit      |
| SH.ANDN.T  | Shift AND-NOT bit  |
| SH.OR.T    | Shift OR bit       |
| SH.NOR.T   | Shift NOR bit      |
| SH.XOR.T   | Shift XOR bit      |
| SH.XNOR.T  | Shift XNOR bit     |
| SH.NAND.T  | Shift NAND bit     |
| SH.ORN.T   | Shift OR-NOT bit   |
| AND.AND.T  | AND-AND bit        |
| AND.ANDN.T | AND-AND-NOT bit    |
| AND.OR.T   | AND-OR bit         |
| AND.NOR.T  | AND-NOR bit        |
| OR.AND.T   | OR-AND bit         |
| OR.ANDN.T  | OR-AND-NOT bit     |
| OR.OR.T    | OR-OR bit          |
| OR.NOR.T   | OR-NOR bit         |
| INS.T      | Insert bit         |
| INSN.T     | Insert negated bit |

## 12. Extract / Insert / Mask

| Mnemonic | Description                |
|----------|----------------------------|
| EXTR     | Extract bit field          |
| EXTR.U   | Extract unsigned bit field |
| DEXTR    | Dual extract               |
| INSERT   | Insert bit field           |
| IMASK    | Create insert mask         |

## 13. Load Instructions

| Mnemonic | Addressing Modes                                        |
|----------|---------------------------------------------------------|
| LD.B     | ABS, BO (base+off, pre/post inc, bit-reverse, circular) |
| LD.BU    | ABS, BO, SR (short)                                     |
| LD.H     | ABS, BO                                                 |
| LD.HU    | ABS, BO                                                 |
| LD.W     | ABS, BO, SR (short)                                     |
| LD.D     | ABS, BO                                                 |
| LD.A     | ABS, BO, SR (short)                                     |
| LD.DA    | ABS, BO                                                 |
| LD.Q     | ABS, BO                                                 |
| LD.DD    | BO                                                      |
| LEA      | ABS, BO (load effective address)                        |
| LHA      | ABS (load high address)                                 |

## 14. Store Instructions

| Mnemonic | Addressing Modes                                        |
|----------|---------------------------------------------------------|
| ST.B     | ABS, BO (base+off, pre/post inc, bit-reverse, circular) |
| ST.H     | ABS, BO                                                 |
| ST.W     | ABS, BO, SR (short)                                     |
| ST.D     | ABS, BO                                                 |
| ST.A     | ABS, BO, SR (short)                                     |
| ST.DA    | ABS, BO                                                 |
| ST.Q     | ABS, BO                                                 |
| ST.DD    | BO                                                      |
| ST.T     | ABS (store bit)                                         |

## 15. Atomic / Swap Operations

| Mnemonic  | Description              |
|-----------|--------------------------|
| SWAP.W    | Atomic swap word         |
| SWAPMSK.W | Atomic swap with mask    |
| CMPSWAP.W | Atomic compare-and-swap  |
| LDMST     | Atomic load-modify-store |

## 16. Cache Operations

| Mnemonic     | Description                              |
|--------------|------------------------------------------|
| CACHEI.I     | Cache invalidate instruction             |
| CACHEI.W     | Cache invalidate data (write)            |
| CACHEI.WI    | Cache invalidate data (write+invalidate) |
| CACHEI.I.VM  | Cache invalidate instruction (virtual)   |
| CACHEI.W.VM  | Cache invalidate data (virtual)          |
| CACHEI.WI.VM | Cache invalidate data (virtual)          |
| CACHEA.I     | Cache allocate instruction               |
| CACHEA.W     | Cache allocate data (write)              |
| CACHEA.WI    | Cache allocate data (write+invalidate)   |
| CACHEA.I.VM  | Cache allocate instruction (virtual)     |
| CACHEA.W.VM  | Cache allocate data (virtual)            |
| CACHEA.WI.VM | Cache allocate data (virtual)            |

## 17. Branch -- Unconditional

| Mnemonic | Format  | Description            |
|----------|---------|------------------------|
| J        | B / SB  | Jump (24-bit / 8-bit)  |
| JA       | B       | Jump absolute          |
| JL       | B       | Jump and link          |
| JLA      | B       | Jump and link absolute |
| JI       | RR      | Jump indirect          |
| JLI      | RR      | Jump and link indirect |
| JRI      | RR      | Jump relative indirect |
| CALL     | B / SB  | Call (32-bit / 16-bit) |
| CALLA    | B       | Call absolute          |
| CALLI    | RR / SR | Call indirect          |
| FCALL    | B       | Fast call              |
| FCALLA   | B       | Fast call absolute     |
| FCALLI   | RR      | Fast call indirect     |
| LOOP     | BRR     | Loop                   |
| LOOPU    | BRR     | Loop unconditional     |

## 18. Branch -- Conditional on Register

| Mnemonic | Description                    |
|----------|--------------------------------|
| JEQ      | Jump if equal                  |
| JNE      | Jump if not equal              |
| JLT      | Jump if less than              |
| JLT.U    | Jump if less than unsigned     |
| JGE      | Jump if greater/equal          |
| JGE.U    | Jump if greater/equal unsigned |
| JZ       | Jump if zero                   |
| JNZ      | Jump if not zero               |
| JLEZ     | Jump if less/equal zero        |
| JLTZ     | Jump if less than zero         |
| JGTZ     | Jump if greater than zero      |
| JGEZ     | Jump if greater/equal zero     |
| JNEI     | Jump if not-equal, increment   |
| JNED     | Jump if not-equal, decrement   |

## 19. Branch -- Conditional on Address/Bit

| Mnemonic | Description               |
|----------|---------------------------|
| JZ.A     | Jump if address is zero   |
| JNZ.A    | Jump if address not zero  |
| JEQ.A    | Jump if address equal     |
| JNE.A    | Jump if address not equal |
| JZ.T     | Jump if bit is zero       |
| JNZ.T    | Jump if bit is not zero   |

## 20. Multiply

| Mnemonic | Description                      |  |
|----------|----------------------------------|--|
| MUL      | Multiply (32x32->32 / 32x32->64) |  |
| MUL.U    | Multiply unsigned (32x32->64)    |  |
| MULS     | Multiply saturated               |  |
| MULS.U   | Multiply saturated unsigned      |  |
| MUL.H    | Multiply packed halfwords        |  |
| MULM.H   | Multiply halfwords (merged)      |  |
| MULR.H   | Multiply halfwords (rounded)     |  |
| MUL.Q    | Multiply Q-format                |  |
| MULR.Q   | Multiply Q-format rounded        |  |
| MULP.B   | Multiply polynomial bytes        |  |

## 21. Multiply-Accumulate (MADD)

| Mnemonic   | Description                                             |
|------------|---------------------------------------------------------|
| MADD       | Multiply-add (32-bit / 64-bit)                          |
| MADD.U     | Multiply-add unsigned                                   |
| MADDS      | Multiply-add saturated                                  |
| MADDS.U    | Multiply-add saturated unsigned                         |
| MADD.H     | Multiply-add halfwords                                  |
| MADDM.H    | Multiply-add halfwords (merged)                         |
| MADDMS.H   | Multiply-add halfwords (merged saturated)               |
| MADDS.H    | Multiply-add halfwords saturated                        |
| MADDR.H    | Multiply-add halfwords rounded                          |
| MADDRS.H   | Multiply-add halfwords rounded saturated                |
| MADD.Q     | Multiply-add Q-format                                   |
| MADDR.Q    | Multiply-add Q-format rounded                           |
| MADDRS.Q   | Multiply-add Q-format rounded saturated                 |
| MADDS.Q    | Multiply-add Q-format saturated                         |
| MADDSU.H   | Multiply-add signed-unsigned halfword                   |
| MADDSUM.H  | Multiply-add signed-unsigned halfword merged            |
| MADDSUMS.H | Multiply-add signed-unsigned halfword merged saturated  |
| MADDSUR.H  | Multiply-add signed-unsigned halfword rounded           |
| MADDSURS.H | Multiply-add signed-unsigned halfword rounded saturated |
| MADDSUS.H  | Multiply-add signed-unsigned halfword saturated         |

## 22. Multiply-Subtract (MSUB)

| Mnemonic   | Description                                       |
|------------|---------------------------------------------------|
| MSUB       | Multiply-subtract (32-bit / 64-bit)               |
| MSUB.U     | Multiply-subtract unsigned                        |
| MSUBS      | Multiply-subtract saturated                       |
| MSUBS.U    | Multiply-subtract saturated unsigned              |
| MSUB.H     | Multiply-subtract halfwords                       |
| MSUBM.H    | Multiply-subtract halfwords merged                |
| MSUBMS.H   | Multiply-subtract halfwords merged saturated      |
| MSUBS.H    | Multiply-subtract halfwords saturated             |
| MSUBR.H    | Multiply-subtract halfwords rounded               |
| MSUBRS.H   | Multiply-subtract halfwords rounded saturated     |
| MSUB.Q     | Multiply-subtract Q-format                        |
| MSUBR.Q    | Multiply-subtract Q-format rounded                |
| MSUBRS.Q   | Multiply-subtract Q-format rounded saturated      |
| MSUBS.Q    | Multiply-subtract Q-format saturated              |
| MSUBAD.H   | Multiply-subtract add halfwords                   |
| MSUBADM.H  | Multiply-subtract add halfwords merged            |
| MSUBADMS.H | Multiply-subtract add halfwords merged saturated  |
| MSUBADS.H  | Multiply-subtract add halfwords saturated         |
| MSUBADR.H  | Multiply-subtract add halfwords rounded           |
| MSUBADRS.H | Multiply-subtract add halfwords rounded saturated |

## 23. Division / Remainder

| Mnemonic  | Description                     |
|-----------|---------------------------------|
| DIV       | Divide signed                   |
| DIV.U     | Divide unsigned                 |
| DIV64     | Divide 64-bit signed            |
| DIV64.U   | Divide 64-bit unsigned          |
| REM64     | Remainder 64-bit signed         |
| REM64.U   | Remainder 64-bit unsigned       |
| DVINIT    | Division init                   |
| DVINIT.B  | Division init byte              |
| DVINIT.BU | Division init byte unsigned     |
| DVINIT.H  | Division init halfword          |
| DVINIT.HU | Division init halfword unsigned |
| DVINIT.U  | Division init unsigned          |
| DVSTEP    | Division step                   |
| DVSTEP.U  | Division step unsigned          |
| DVADJ     | Division adjust                 |

## 24. Floating Point -- Single Precision (32-bit)

| Mnemonic | Description                         |
|----------|-------------------------------------|
| ADD.F    | FP add                              |
| SUB.F    | FP subtract                         |
| MUL.F    | FP multiply                         |
| DIV.F    | FP divide                           |
| MADD.F   | FP multiply-add                     |
| MSUB.F   | FP multiply-subtract                |
| ABS.F    | FP absolute                         |
| NEG.F    | FP negate                           |
| CMP.F    | FP compare                          |
| MAX.F    | FP maximum                          |
| MIN.F    | FP minimum                          |
| QSEED.F  | FP reciprocal seed                  |
| ITOF     | Integer to float                    |
| UTOF     | Unsigned to float                   |
| FTOI     | Float to integer                    |
| FTOIZ    | Float to integer (round to zero)    |
| FTOIN    | Float to integer (round to nearest) |
| FTOU     | Float to unsigned                   |
| FTOUZ    | Float to unsigned (round to zero)   |
| FTOQ31   | Float to Q31                        |
| FTOQ31Z  | Float to Q31 (round to zero)        |
| Q31TOF   | Q31 to float                        |
| FTOHP    | Float to half-precision             |
| HPTOF    | Half-precision to float             |
| FTODF    | Float to double                     |

## 25. Floating Point -- Double Precision (64-bit)

| Mnemonic | Description                             |
|----------|-----------------------------------------|
| ADD.DF   | DP add                                  |
| SUB.DF   | DP subtract                             |
| MUL.DF   | DP multiply                             |
| DIV.DF   | DP divide                               |
| MADD.DF  | DP multiply-add                         |
| MSUB.DF  | DP multiply-subtract                    |
| ABS.DF   | DP absolute                             |
| NEG.DF   | DP negate                               |
| CMP.DF   | DP compare                              |
| MAX.DF   | DP maximum                              |
| MIN.DF   | DP minimum                              |
| QSEED.DF | DP reciprocal seed                      |
| ITODF    | Integer to double                       |
| UTODF    | Unsigned to double                      |
| LTODF    | Long to double                          |
| ULTODF   | Unsigned long to double                 |
| DFTOI    | Double to integer                       |
| DFTOIZ   | Double to integer (round to zero)       |
| DFTOIN   | Double to integer (round to nearest)    |
| DFTOU    | Double to unsigned                      |
| DFTOUZ   | Double to unsigned (round to zero)      |
| DFTOL    | Double to long                          |
| DFTOLZ   | Double to long (round to zero)          |
| DFTOUL   | Double to unsigned long                 |
| DFTOULZ  | Double to unsigned long (round to zero) |
| DFTOF    | Double to float                         |

## 26. Bit Manipulation / CRC / Misc

| Mnemonic | Description               |
|----------|---------------------------|
| BMERGE   | Bit merge                 |
| BSPLIT   | Bit split                 |
| PACK     | Pack                      |
| UNPACK   | Unpack                    |
| PARITY   | Parity                    |
| POPCNT.W | Population count          |
| CRC32.B  | CRC-32 byte               |
| CRC32B.W | CRC-32 byte-wise word     |
| CRC32L.W | CRC-32 long word          |
| CRCN     | CRC-N configurable        |
| IXMIN    | Index of minimum          |
| IXMIN.U  | Index of minimum unsigned |
| IXMAX    | Index of maximum          |
| IXMAX.U  | Index of maximum unsigned |

## 27. Core Register Access

| Mnemonic | Description                  |
|----------|------------------------------|
| MFCR     | Move from core register      |
| MTCR     | Move to core register        |
| MFDCR    | Move from data core register |
| MTDCR    | Move to data core register   |

---

## Summary by Category

| #  | Category                         | Count |
|----|----------------------------------|-------|
| 1  | System / Special                 | 28    |
| 2  | Move / Load Immediate            | 9     |
| 3  | Address Register                 | 11    |
| 4  | Integer Arithmetic (Word)        | 24    |
| 5  | Integer Arithmetic (Byte)        | 14    |
| 6  | Integer Arithmetic (Halfword)    | 20    |
| 7  | Comparison / Conditional         | 15    |
| 8  | Conditional Bit (AND/OR/XOR+cmp) | 24    |
| 9  | Logical                          | 8     |
| 10 | Shift / Rotate / Count           | 12    |
| 11 | Bit Operations (individual)      | 26    |
| 12 | Extract / Insert / Mask          | 5     |
| 13 | Load                             | 12    |
| 14 | Store                            | 10    |
| 15 | Atomic / Swap                    | 4     |
| 16 | Cache                            | 12    |
| 17 | Branch (Unconditional)           | 15    |
| 18 | Branch (Conditional Register)    | 14    |
| 19 | Branch (Conditional Addr/Bit)    | 6     |
| 20 | Multiply                         | 10    |
| 21 | MADD                             | 20    |
| 22 | MSUB                             | 20    |
| 23 | Division / Remainder             | 15    |
| 24 | FP Single Precision              | 25    |
| 25 | FP Double Precision              | 26    |
| 26 | Bit Manip / CRC / Misc           | 14    |
| 27 | Core Register Access             | 4     |
|    | **TOTAL**                        |**~400**|

---

## Missing / Unsupported Opcodes

The following TriCore ISA instructions are **not implemented** in the simulator.
They are intentionally omitted because they relate to hardware features that have
no meaningful behavior in a software simulation context.

## MMU / TLB Instructions

| Mnemonic    | Description                    | Reason                            |
|-------------|--------------------------------|-----------------------------------|
| TLBFLUSH.A  | Flush TLB (all)                | No MMU/TLB model in simulator     |
| TLBFLUSH.B  | Flush TLB (block)              | No MMU/TLB model in simulator     |
| TLBMAP      | Map TLB entry                  | No MMU/TLB model in simulator     |
| TLBDEMAP    | Demap TLB entry                | No MMU/TLB model in simulator     |
| TLBPROBE.A  | Probe TLB by address           | No MMU/TLB model in simulator     |
| TLBPROBE.I  | Probe TLB by index             | No MMU/TLB model in simulator     |
| LDTLB       | Load TLB entry from registers  | No MMU/TLB model in simulator     |

## Hardware Debug Instructions

| Mnemonic | Description                   | Reason                             |
|----------|-------------------------------|------------------------------------|
| BREAK    | Hardware breakpoint (CoreSight)| Use `@bp` parser command instead  |

## Coprocessor / Safety Extensions

| Mnemonic | Description                     | Reason                            |
|----------|---------------------------------|-----------------------------------|
| CRC32.W  | CRC-32 word (TC1.6.2 variant)  | CRC32B.W / CRC32L.W cover this   |
| DIV.E    | Division extended (Aurix TC4x) | TC4x-specific, not targeted       |

## Notes

- **DEBUG** instruction is decoded but executes as NOP (real HW enters JTAG debug mode).
- **WAIT** instruction is decoded but executes as NOP (no power management model).
- **DISABLE/ENABLE/RESTORE** are decoded; they modify ICR.IE internally.
- All **CACHE** instructions are decoded but execute as NOP (no cache model -- memory is flat).
- **SVLCX/RSLCX/STLCX/STUCX/LDLCX/LDUCX** are fully implemented with context save area management.

