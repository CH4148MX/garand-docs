---
title: Garand Architecture
author: Ibrahima Keita, Dung Nguyen, Sergio Ly
---

<!-- # Garand Architecture -->

## Descriptions

Named Garand Architecture. As a Special Purpose architecture derived from RISC/ARM, its primary support is gaming and graphics processing. It contains low-level graphics support, compatible with openGL. Multiple co-processing is also possible. Vectorization such as Fused Multiplacation-Add is included. On Developer side, extensible interface is documented as Add-On.

Word size is 64 bits, while the architechture supports integer and fixed-point number. There will be 16 general purpose registers, 16 I/O registers, Link Register, Stack Pointer register. For each instructions, 3 is the maximum number of operands. Fetching model is multiple instruction per word - Princeton (unified instruction and data memory), with 48-bit Word addressing.

Supported addressing mode are:

-   Load-store: for memory load/store instructions only
-   Register, Immediate, PC + Immediate Offset (PC Relative): for the rest of the instructions

## OpCode model:

| Flag (8 bits) | OpCode/Type (8 bits) | Register/Immediate reference (48 bits) |
| ------------- | -------------------- | -------------------------------------- |

## Instructions documentation by types:

### Load/Store

#### MREAD

Read from memory address indicated by register S, offset by register T.

#### MRITE

Write to memory address indicated by register S, offset by register T

#### BIND

Bind a IO register to a specific device. Any write/read instruction will be automatically forwarded to the IO device.

### Control

#### BRUHCC

Change the Instruction Pointer to address indicated by register S.

##### CC Code

-   Always
-   unsigned less than
-   signed less than
-   unsigned greater than
-   signed greater than
-   unsigned greater than or equal
-   signed greater than or equal
-   unsigned less than or equal
-   signed less than or equal
-   Equal/Zero
-   Inequal/Not-Zero
-   Overflow
-   No overflow
-   Negative
-   Positive or zero

##### Branch target

Support both relative/absolute jump. subroutine: relative. Function call: absolute.

### Integer

#### ADD

Add two registers S and T. Store in register X. Overflow is checked.

#### MUL

Mutiply two registers S and T. Store in register X.

#### DIV

Divide two registers S and T. Store in register X.

#### AND

Bitwise And two registers S and T. Store in register X.

#### NAND

Bitwise Nand two registers S and T. Store in register X.

#### OR

Bitwise Or two registers S and T. Store in register X.

#### XOR

Bitwise Xor two registers S and T. Store in register X.

#### NOT

Bitwise Not register S. Store in register X.

#### FLIP

Flip bits of register S and store in register X.

#### ASR

Arithmetic shift right Register S by Register T amount, stores in Register X.

#### LSR

Logical shift right Register S by Register T amount, stores in Register X.

#### LSL

Logical shift left Register S by Register T amount, stores in Register X.

#### RSR

Rotational shift right Register S by Register T amount, stores in Register X.

#### RSL

Rotational shift left Register S by Register T amount, stores in Register X.

### Fixed-point number

Use the same mnemonics as general purpose registers'. It is undefined behaviour to mix two types of registers in the instructions.

### Special

#### BREAK

Break execution. In the simulator, the processor must pause processing after this point.

#### HALT

Stop all execution. In the simulator, the processor must stop processing at this point and exit the program.
