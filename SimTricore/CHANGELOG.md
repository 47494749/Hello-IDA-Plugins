# Changelog -- SimTricore

All notable changes to the TriCore Simulator IDA plugin.

---

## [Current] -- 2026-07-23

### Features

- Full TriCore ISA simulation with ~400 unique mnemonics (~898 encoding variants)
- Comment-driven command parser (`@commands` in IDA repeatable comments)
- Register assignment with generator expressions (`@d0 = rand(0, 0xFF)`)
- Breakpoints: PC-based, register-conditional, memory-conditional
- Memory write commands (single, list, range fill) for 8/16/32/64-bit
- Memory dump to IDA output window (8/16/32/64-bit hex)
- Mock read system with multi-phase sequential values and loop modifiers
- Save flags: `@save_trace`, `@save_memory_trace`, `@save_memory_map`, `@save_fuzz_results`
- Fuzzing engine with multi-step iteration, per-step generators, and stop conditions
- Generator system: `rand`, `list`, `range`, `edges`, `signed_edges`, `pow2`, `pow2_minus1`, `walk`, `rot_walk`, `byte_walk`, `flip`, `neg`, `mirror`, `arith`, `off`, `aligned`, `overflow`, `underflow`, `stack`, `flag`
- `op_*` shortcuts: `op_add`, `op_sub`, `op_mul`, `op_div`, `op_mod`, `op_and`, `op_or`, `op_xor`, `op_not`, `op_shl`, `op_shr`, `op_asl`, `op_asr`, `op_abs`, `op_clz`
- Multi-core support (`@cpu0` .. `@cpu6`) for AURIX targets
- Infinite-loop detection with configurable threshold (`@loop_limit`)
- Simulation timeout (`@timeout`)
- Max instruction count (`@max`)
- Debug mode (`@debug`) with per-instruction trace logging
- Full context save/restore (SVLCX/RSLCX/STLCX/STUCX/LDLCX/LDUCX)
- Floating-point: single-precision (32-bit) and double-precision (64-bit)
- CRC instructions: CRC32.B, CRC32B.W, CRC32L.W, CRCN
- Trap system (TRAPV, TRAPSV, TRAPINV) with class/TIN recording
- IDA segment loading into flat memory buffer
- Background thread execution with UI progress bar
- Batch mode (`-A` flag) for headless/idalib operation
- Output viewer (custom IDA viewer)
- Memory access tracing (address, size, R/W, privilege, PC)
- Register snapshot tracing (all D/A registers + PSW per step)

### Opcode Coverage

| Category                         | Count |
|----------------------------------|-------|
| System / Special                 | 28    |
| Move / Load Immediate            | 9     |
| Address Register                 | 11    |
| Integer Arithmetic (Word)        | 24    |
| Integer Arithmetic (Byte)        | 14    |
| Integer Arithmetic (Halfword)    | 20    |
| Comparison / Conditional         | 15    |
| Conditional Bit (AND/OR/XOR+cmp) | 24    |
| Logical                          | 8     |
| Shift / Rotate / Count           | 12    |
| Bit Operations (individual)      | 26    |
| Extract / Insert / Mask          | 5     |
| Load                             | 12    |
| Store                            | 10    |
| Atomic / Swap                    | 4     |
| Cache (NOP in simulation)        | 12    |
| Branch (Unconditional)           | 15    |
| Branch (Conditional Register)    | 14    |
| Branch (Conditional Addr/Bit)    | 6     |
| Multiply                         | 10    |
| MADD                             | 20    |
| MSUB                             | 20    |
| Division / Remainder             | 15    |
| FP Single Precision              | 25    |
| FP Double Precision              | 26    |
| Bit Manip / CRC / Misc           | 14    |
| Core Register Access             | 4     |
| **TOTAL**                        |**~400**|

### Not Implemented (by design)

- MMU/TLB instructions (TLBFLUSH, TLBMAP, TLBDEMAP, TLBPROBE, LDTLB)
- BREAK (hardware debug -- use `@bp` instead)
- CRC32.W (TC1.6.2 variant -- covered by CRC32B.W / CRC32L.W)
- DIV.E (TC4x-specific)
