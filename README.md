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

### TriCore Simulator -- IDA Pro Plugin (Prototype)

> **[!] This is a prototype / work-in-progress.**
> Features marked as *not yet implemented* are planned but not available in the current build.

A lightweight **Infineon TriCore** CPU simulator embedded inside **IDA Pro**.  
It allows you to simulate individual functions directly from the disassembly view, inspect register and memory state, intercept memory reads with mock values, and perform basic fuzzing over register and memory inputs -- all driven by structured commands written as **repeatable comments at the top of the target function**.


## Notes
The UI might have some omissions (I do not enjoy creating UIs, and since the program was originally just for me, I left several things out). However, the program should not cause any issues.

## Bug Reports and Suggestions
If you find any bugs (which there likely are) or have any suggestions, you can contact me here or on LinkedIn.


