# Changelog -- SimAArch64

All notable changes to the AArch64 Simulator IDA plugin.

---

## [Current] -- 2026-07-23

### Features

- Full ARMv8-A (AArch64/ARM64) ISA simulation with ~300+ unique mnemonics
- ARMv8.0 base + ARMv8.1 LSE atomics + ARMv8.2 FP16/DotProd + ARMv8.4/8.5 flag manipulation
- Comment-driven command parser (`@commands` in IDA repeatable comments)
- Register assignment with generator expressions (`@x0 = rand(0, 0xFF)`)
- Breakpoints: PC-based, register-conditional (X/W), memory-conditional
- Memory write commands (single, list, range fill) for 8/16/32/64-bit
- Memory dump to IDA output window (8/16/32/64-bit hex)
- Mock read system with multi-phase sequential values and loop modifiers
- Save flags: `@save_trace`, `@save_memory_trace`, `@save_memory_map`, `@save_fuzz_results`
- Fuzzing engine with multi-step iteration, per-step generators, stop conditions, and log types
- Generator system: `rand`, `list`, `range`, `edges`, `signed_edges`, `pow2`, `pow2_minus1`, `walk`, `rot_walk`, `byte_walk`, `flip`, `neg`, `mirror`, `arith`, `off`, `aligned`, `overflow`, `underflow`, `stack`, `flag`
- `op_*` shortcuts: `op_add`, `op_sub`, `op_mul`, `op_div`, `op_mod`, `op_and`, `op_or`, `op_xor`, `op_not`, `op_shl`, `op_shr`, `op_asl`, `op_asr`, `op_abs`, `op_clz`
- Infinite-loop detection with configurable threshold (`@loop_limit`)
- Simulation timeout (`@timeout`, default 30000 ms)
- Max instruction count (`@max`)
- Debug mode (`@debug`) with per-instruction trace logging
- Full EL0-EL3 exception model with VBAR-based vector dispatch
- SPSR/ELR/ESR save on exception
- SIMD/NEON: full 128-bit vector register file (V0-V31) with scalar views (B/H/S/D/Q)
- Floating-point: half-precision (FP16), single-precision, double-precision
- Cryptography extensions: AES (AESE/AESD/AESMC/AESIMC), SHA-1, SHA-256
- Dot product instructions (SDOT, UDOT)
- LSE atomics (LDADD, LDCLR, LDSET, LDEOR, SWP, CAS, CASP)
- CRC32 instructions (CRC32B/H/W/X, CRC32CB/CH/CW/CX)
- Flag manipulation (CFINV, SETF8, SETF16, RMIF)
- Auto SP init (if SP==0, auto-initialized to middle of memory buffer, 16-byte aligned)
- Auto LR init (X30 set to end_addr so RET naturally terminates simulation)
- 16 MiB flat memory buffer with 64-bit address translation
- IDA segment loading into flat memory buffer
- Background thread execution with UI progress bar
- Batch mode (`-A` flag) for headless/idalib operation
- Output viewer (custom IDA viewer)
- Memory access tracing (PC, address, value, size, R/W, privileged)
- Register snapshot tracing (all X0-X30 + SP + PC + PSTATE per step)

### Opcode Coverage

| Category                       | Count |
|--------------------------------|-------|
| Data Processing (Immediate)    | 10    |
| Data Processing (Register)     | 10    |
| Shift / Bitfield / Extract     | 8     |
| Multiply / Divide              | 10    |
| Load / Store (Single)          | 10    |
| Load / Store Pair              | 4     |
| Load / Store Exclusive/Ordered | 9     |
| Branch                         | 8     |
| Conditional Select             | 4     |
| Conditional Compare            | 2     |
| System / Hints                 | 11    |
| Exception Generation           | 8     |
| Floating-Point Scalar          | 20+   |
| Floating-Point Conversion      | 10+   |
| SIMD (NEON) Basic              | 30+   |
| Bit Manipulation               | 6     |
| SIMD Advanced                  | 50+   |
| LSE Atomics                    | 18    |
| CRC32                          | 8     |
| Half-Precision FP              | 20+   |
| Flag Manipulation              | 4     |
| SIMD By-Element                | 12+   |
| Dot Product                    | 2     |
| Cryptography (AES/SHA)         | 14    |
| **TOTAL**                      |**~300+**|
