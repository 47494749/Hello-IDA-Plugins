# Changelog -- SimArm

All notable changes to the ARM Simulator IDA plugin.

---

## [Current] -- 2026-07-23

### Features

- Full ARMv7-A (AArch32) ISA simulation with ~250 unique mnemonics
- ARM (A32) and Thumb (T16/T32) dual-mode support with auto-detection from IDA's `T` segment register
- Comment-driven command parser (`@commands` in IDA repeatable comments)
- Register assignment with generator expressions (`@r0 = rand(0, 0xFF)`)
- Breakpoints: PC-based, register-conditional, memory-conditional
- Memory write commands (single, list, range fill) for 8/16/32/64-bit
- Memory dump to IDA output window (8/16/32/64-bit hex)
- Mock read system with multi-phase sequential values and loop modifiers
- Save flags: `@save_trace`, `@save_memory_trace`, `@save_memory_map`, `@save_fuzz_results`
- Fuzzing engine with multi-step iteration, per-step generators, stop conditions, and log types
- Generator system: `rand`, `list`, `range`, `edges`, `signed_edges`, `pow2`, `pow2_minus1`, `walk`, `rot_walk`, `byte_walk`, `flip`, `neg`, `mirror`, `arith`, `off`, `aligned`, `overflow`, `underflow`, `stack`, `flag`
- `op_*` shortcuts: `op_add`, `op_sub`, `op_mul`, `op_div`, `op_mod`, `op_and`, `op_or`, `op_xor`, `op_not`, `op_shl`, `op_shr`, `op_asl`, `op_asr`, `op_abs`, `op_clz`
- Infinite-loop detection with configurable threshold (`@loop_limit`)
- Simulation timeout (`@timeout`, default 30000 ms)
- Max instruction count (`@max`, default 100000)
- Debug mode (`@debug`) with per-instruction trace logging
- IT block state machine for Thumb-2 If-Then execution
- Barrel shifter with all 5 shift types (LSL, LSR, ASR, ROR, RRX)
- All 15 ARM condition codes
- VFP/NEON coprocessor: single/double-precision FP, NEON integer/logical/shift
- Exception vectors at 0x00-0x1C (Reset, Undef, SVC, Prefetch Abort, Data Abort, Hyp, IRQ, FIQ)
- Big-endian support (detected from IDA database)
- Call-depth tracking with shadow LR stack for BL/BLX subroutine calls
- 16 MiB flat memory buffer with configurable base address
- IDA segment loading into flat memory buffer
- Background thread execution with UI progress bar
- Batch mode (`-A` flag) for headless/idalib operation
- Output viewer (custom IDA viewer)
- Memory access tracing (address, size, R/W, privilege, PC)
- Register snapshot tracing (all GPRs + CPSR per step)

### Opcode Coverage

| Category                    | Count |
|-----------------------------|-------|
| Data Processing             | 23    |
| Multiply                    | 8     |
| Load / Store (Single)       | 20    |
| Load / Store Multiple       | 4     |
| Branch                      | 10    |
| Status Register Access      | 3     |
| VFP / NEON Coprocessor      | 56+   |
| Miscellaneous / Hints       | 12    |
| Saturating Arithmetic       | 8     |
| Parallel Add/Sub (SIMD)     | 36    |
| Packing/Unpacking/Reversal  | 18+   |
| Signed Multiply (DSP)       | 30    |
| Divide                      | 2     |
| Bit Field                   | 4     |
| Exception / System          | 8     |
| Thumb-Specific              | 6     |
| **TOTAL**                   |**~250**|
