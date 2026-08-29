# Changelog -- SBT Arm

All notable changes to the ARM Static Binary Translation IDA plugin.

---

## [0.1] -- 2026-08-29

### Features

- Full ARMv7-A (AArch32) static binary translation to compilable C source code
- ARM (A32), Thumb (T16), and Thumb-2 (T32) instruction set support with auto-detection from IDA's `T` segment register
- Per-byte code mode tracking for mixed ARM/Thumb functions
- 5-level optimization pipeline: raw goto/labels (0) through advanced control flow with switch/return/inline (5)
- Dual output: optimized (`_sbt.c`) and non-optimized (`_sbt_noopt.c`) C files
- Header file generation (`_sbt.h`) with memory region externs, global variable `#define` macros, and function prototypes
- Recursive call-graph crawl (BFS, up to 16 levels deep) with `follow_calls` option
- Indirect jump/call resolution via IDA cross-references
- Region-based memory model with automatic IDA segment classification (Flash, SRAM, MMIO)
- ARM Cortex-M address map heuristics for region classification
- Flash/ROM pre-initialization from IDA database via `sbt_init_data()`
- MMIO stub generation for peripheral addresses
- Flat memory fallback (16 MiB buffer) when no IDA segments are available
- Known library function replacement (memcpy, memmove, memset, memcmp, strlen, strcmp, strncmp, strcpy, strncpy, strcat, strchr, rand, srand, abs, malloc, free, calloc, realloc, printf, sprintf, snprintf)
- Global variable alias collection from IDA's name list
- Full NZCV flag tracking for arithmetic, logical, compare, and barrel shifter operations
- All 14 ARM condition codes via inline macros
- Barrel shifter with all shift modes (LSL, LSR, ASR, ROR) for immediate and register operands
- IT block (If-Then) state machine with CPSR flag preservation for Thumb-2
- VFP single/double-precision floating-point translation (VLDR, VSTR, VADD, VSUB, VMUL, VDIV, VNEG, VABS, VSQRT, VMLA, VMLS, VCMP, VCVT, VMOV, VMRS, VMSR, VPUSH, VPOP, VLDM, VSTM)
- NEON integer/logical/shift translation (VADD.I, VSUB.I, VMUL.I, VAND, VBIC, VORR, VORN, VEOR, VMVN, VSHL, VSHR, VDUP.32)
- DSP/SIMD instructions: parallel add/sub, saturating arithmetic, dual multiply, sum of absolute differences
- Exclusive load/store (LDREX/STREX) with exclusive monitor simulation
- Saturating arithmetic (QADD, QSUB, QDADD, QDSUB, SSAT, USAT)
- Bit field operations (BFI, BFC, SBFX, UBFX)
- Callee-saved register preservation (r4--r11) around inter-function calls
- Inline assembly comments (original ARM mnemonics as C comments)
- Fuzz harness generation: AFL-compatible `main()` (`-DSBT_FUZZ_HARNESS`) and libFuzzer `LLVMFuzzerTestOneInput` (`-DSBT_LIBFUZZER`)
- Batch mode support (`-A` flag) for headless/idalib operation
- Self-contained output: ARM runtime support library embedded in generated C file

### Opcode Coverage

| Category                         | Count |
|----------------------------------|-------|
| Thumb (16-bit)                   | 40+   |
| Thumb-2 (32-bit)                 | 50+   |
| ARM Data Processing              | 18    |
| ARM Multiply                     | 8     |
| ARM Load / Store                 | 20+   |
| ARM Branch                       | 4     |
| ARM Status Register              | 2     |
| ARM Saturating/BitField/Packing  | 16    |
| ARM DSP / SIMD                   | 30+   |
| ARM Divide                       | 2     |
| VFP (Floating-Point)             | 20+   |
| NEON (SIMD)                      | 10+   |
| System / Hints                   | 8     |
| **TOTAL**                        |**~230+**|
