# Hello-IDA-Plugins by Luigi Origa
## Overview

The plugin consists of 6 distinct modules (for both IDA32 and IDA64 for Hex-rays IDA Pro 8 and IDA64 only for Hex-rays IDA Pro 9) that perform various functions:

### Help Module
- Enables or disables other modules or individual functions within the modules.
- Configures shortcuts and menus.
- **Note:** All modules function independently.

### BeepBeep Module
This module implements simple functions for code management:
- **Make Byte/Word/Double/Quad**: Creates all data of the selected type and any offsets/pointers from a selected block in the asm window.
- **Go Top/Bottom**: Moves the cursor to the start or end of the current function.
- **Delete Function**: Deletes the current function.
- **Copy Hex**: Copies the hex values of the current function.

### Search Binary Module
- Searches for values within the file in various formats, including permuting the positions of numeric sequences.

### Search Similar Module
- Searches for functions similar to the current one.
- Allows setting maximum differences and size variations between functions.

### Search XOR Module
- Supports various architectures.
- Searches for how many XOR instructions are present in each function.
- Useful for finding functions involved in encryption.

### Search Tables Module
- Finds values and tables used for hashing, elliptic curves, AES, CRC, etc.

## Notes
The UI might have some omissions (I do not enjoy creating UIs, and since the program was originally just for me, I left several things out). However, the program should not cause any issues.

## Bug Reports and Suggestions
If you find any bugs (which there likely are) or have any suggestions, you can contact me here or on LinkedIn.


# TriCore Simulator — IDA Pro Plugin (Prototype)

> **⚠️ This is a prototype / work-in-progress.**
> Features marked as *not yet implemented* are planned but not available in the current build.

A lightweight **Infineon TriCore** CPU simulator embedded inside **IDA Pro**.  
It allows you to simulate individual functions directly from the disassembly view, inspect register and memory state, intercept memory reads with mock values, and perform basic fuzzing over register and memory inputs — all driven by structured commands written as **repeatable comments at the top of the target function**.

---

## Table of Contents

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

---

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

---

## Register Assignment

Set the initial CPU state before simulation starts. All registers support a **value** or a **generator expression** (see [Generators](#generators)).

```
@d0..@d15  = value|gen     ; Set data register D0–D15
@a0..@a15  = value|gen     ; Set address register A0–A15
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

Commands execute in the order they appear. `@run` triggers the simulator. If no `@run` is present, an implicit run is appended automatically. Multiple `@run` commands in sequence resume emulation from the current PC.

Save flags (`@save_trace`, `@save_memory_trace`, `@save_memory_map`, `@save_fuzz_results`) are always deferred to the very end regardless of where they appear in the comment block.

| Command | Description |
|---------|-------------|
| `@run` | Run or resume the simulator |
| `@bp <addr>` | Add a breakpoint at the given address |
| `@bp Dn op value` | Break when data register `Dn` satisfies condition |
| `@bp An op value` | Break when address register `An` satisfies condition |
| `@bp memN[addr] op val` | Break on memory condition (`N` = 8, 16, 32, 64) |
| `@max <N>` | Maximum number of instructions to execute |
| `@timeout <ms>` | Simulation timeout in milliseconds |
| `@loop_limit <N>` | Infinite-loop detection check interval (default: 1000) |

Supported breakpoint operators: `==`  `!=`  `>`  `<`  `>=`  `<=`

**Examples:**

```
; @bp 0x80013FFC
; @bp D5 >= 0x100
; @bp A3 == 0xD0001000
; @bp mem32[0xD0002000] != 0
; @max 10000
; @timeout 500
; @loop_limit 2000
; @run
```

---

## CPU Core Selection

Select which TriCore CPU core to target for simulation.

```
@cpu0 .. @cpu6
```

**Example:**

```
; @cpu0
; @d0 = 1
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

Generators are expressions that produce values dynamically. They can be used wherever a plain numeric value is accepted, unless noted otherwise.

### Group 1 — Scalars *(registers, memory, fuzz)*

| Generator | Description |
|-----------|-------------|
| `rand(min, max)` | Random value uniformly drawn from `[min, max]` |
| `list(v1, v2, ...)` | In single-shot mode: uses the first value; in fuzz mode: iterates through values |
| `<numeric>` | Plain literal value (decimal or hex) |

### Group 2 — Register-relative *(D/A registers, memory, fuzz)*

| Generator | Description |
|-----------|-------------|
| `neg(Dn\|An)` | Arithmetic negation of the register value |
| `mirror(Dn\|An)` | Bit-reversal of the register value |
| `arith(Dn\|An, op, val)` | Arithmetic operation on register: `+` `-` `*` `/` `%` `&` `\|` `^` `<<` `>>` |
| `off(An, val)` | Address register value plus a signed offset |
| `aligned(An, n)` | Address register aligned down to `n` bytes |
| `overflow(An)` | Address just past the end of the region pointed to by `An` |
| `underflow(An)` | Address just before the region pointed to by `An` |

### Group 3 — Pointer *(A registers, memory, fuzz)*

| Generator | Description |
|-----------|-------------|
| `stack(offset)` | Stack pointer (`A10`) plus a signed offset |

### Group 4 — Iteration-based *(fuzz only)*

| Generator | Description |
|-----------|-------------|
| `range(min, max, step)` | Iterate from `min` to `max` incrementing by `step` each fuzz iteration |

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
| `mem8[addr]` | 8-bit memory location |
| `mem16[addr]` | 16-bit memory location |
| `mem32[addr]` | 32-bit memory location |
| `mem64[addr]` | 64-bit memory location |

**Example:**

```
; @d0  = 0xD0001000        ; base pointer
; @a4  = 0xD0001000
; @max 2000
; @fuzz
;   0 D1 = range(0, 0xFF, 1)
;   1 D1 = rand(0, 0xFFFFFFFF)
;   2 mem32[0xD0001000] = rand(0, 0xFFFFFFFF)
; @save_fuzz_results
```

- The `step_index` (0, 1, 2, …) controls the order of fuzz steps within one iteration.
- `call(addr)` optionally redirects execution to a different entry point for that step.
- Each step's generator is evaluated fresh for every iteration.
- `@save_fuzz_results` collects register state, PC, and any trap information for each iteration.

---

## Notes

- Commands are **case-sensitive** (`@run` ≠ `@Run`).
- Values are decimal by default; prefix with `0x` for hexadecimal.
- Lines beginning with `;` or `#` inside the comment block are pure comments and are ignored by the parser.
- `@save_*` commands are always deferred and executed last, regardless of position.
- Multiple `@run` commands resume from the current PC rather than restarting from the function entry.
- This is a **prototype**: stability, accuracy, and feature completeness are not guaranteed. Feedback and bug reports are welcome via the issue tracker.
