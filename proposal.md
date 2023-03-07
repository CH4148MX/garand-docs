---
title: Garand Architecture
author: Ibrahima Keita, Dung Nguyen, Sergio Ly
---

<!-- # Garand Architecture -->

## Descriptions
The Garand Architecture is a special-purpose architecture designed with gaming and graphics processing in mind.

<!-- Named Garand Architecture. As a Special Purpose architecture derived from RISC/ARM, its primary support is gaming and graphics processing. It contains low-level graphics support, compatible with openGL. Multiple co-processing is also possible. Vectorization such as Fused Multiplacation-Add is included. On Developer side, extensible interface is documented as Add-On. -->

## Specifications

### Word Size
Our word size is 32 bits, which makes a half word 16 bits, a double word 64 bits, and so on.

### Data Types
In terms of data types, we aim to support two distinct numerical types: integer and fixed point.

#### Integer

The ISA defines two types of integer:
-   Unsigned integer: Value ranges from 0 to the largest positive number can be binary encoded.
    For example, 32 bits unsigned integer ranges from 0 to 4294967295 ($2^{32}-1$).
    Bitwise instructions always treat value as unsigned.
-   Signed integer: The ISA assume two's compliment representation for signed integer.
    For example, 32 bits signed integer ranges from -2147483648($-2^{31}$) to 2147483647 ($2^{31}-1$).
    Most arithmetic instructions supports signed integer.

#### Fixed Point 
In graphics, precision in calculations is fundamental; from raytracing, to interpolation, etc. As such, it is fundamental that, for a graphics-based architecture, we have some sort of mechanism for precise calculations. Therefore, we will be implementing fixed point precision.

For our implementation of fixed point, we will reserve the lower 23 bits of a word to store our _mantissa_, which is the number that comes after the decimal point. Our _exponent_, which is 8 bits, will immediately follow the mantissa in bit order, taking bits 23 through 30 inclusive. Finally, the most significant bit (which is bit 31) will act as a signage bit, allowing us to have positive and negative fixed-point values.

A detailed diagram of this can be seen below.

![Fixed Point Spec](https://media.discordapp.net/attachments/1079631715049426946/1082761338864029777/Untitled_Note_-_Mar_7_2023_3.26_PM.png)

As these numbers require special treatment, we will implement instructions specifically for them.

### Registers
Our architecture is designed to support 36 total registers. The breakdown is as follows:
- 16 General-Purpose Registers (frontend syntax `R##`).
- 16 IO-specific registers (frontend syntax: `I##`)
- 4 special purpose registers:
    - Condition Flag (frontend syntax: `CND`) - <description here>.
    - Program Counter (frontend syntax: `PC`) - Points to the currently executed instruction.
    - Link Register (frontend syntax: `LR`) - Commonly stores an address to a function (used in the caller-callee specification).
    - Stack Pointer (frontend syntax: `SP`) - <description here>.

Each of these registers uses a single word (32 bits) as its underlying data type.

#### Encoding

| Name  | Size | Encoding | Description                    |
| ----- | ---- | -------- | ------------------------------ |
| `R##` | 32   | 0-15     | General-Purpose Registers 0-15 |
| `I##` | 32   | 0-15     | IO-specific Registers 0-15     |
| `SP`  | 32   | 16       | Stack Pointer                  |

### Fetching Model
As described earlier, our word size is 32 bits. To fully take advantage of this, we will use the `multiple words per instruction` fetch model. This will give us optimal code size while performing relatively efficiently.

### Memory Architecture
We want to keep our instruction and data memory together; as such, we will use the _Princeton_ architecture.

### Addressing Modes
- Load-Store: for memory load/store instructions only.
- Register, Immediate, PC + Immediate Offset (PC Relative): for the rest of the instructions.

### Instructions
<Description here>.

#### Encoding

#### Operation Codes
| Flag | OpCode/Type | Register/Immediate reference |
| ---- | ----------- | ---------------------------- |


## Instructions documentation by types:

### Load/Store

#### MREAD

##### Assembler syntax

`MREAD   RX, RM, RN`

##### Description

Read from memory address and store value in register X. The memory address
is calculated by base value from register M, plus 4 times of offset value from
register N.

##### Psuedo C-code

`RX = RM[RN << 2]`

#### MRITE

##### Assembler syntax

`MRITE   RX, RM, RN`

##### Description
Write to memory address using value stored in register X. The memory address
is calculated by base value from register M, plus 4 times offset value from
register N.

##### Psuedo C-code

`RM[RN << 2] = RX`

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

Each instructions has **(I)mmediate** variation, which allow immediate to be written
in-place of register T.

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
