---
title: Garand Architecture
author: Ibrahima Keita, Dung Nguyen, Sergio Ly
geometry: margin=2cm
output: pdf_document
---

<!-- # Garand Architecture -->

## Description
The Garand Architecture is a special-purpose architecture designed with graphics processing in mind. It has hardware level support for graphics rendering.

<!-- Named Garand Architecture. As a Special Purpose architecture derived from RISC/ARM, its primary support is gaming and graphics processing. It contains low-level graphics support, compatible with openGL. Multiple co-processing is also possible. Vectorization such as Fused Multiplacation-Add is included. On Developer side, extensible interface is documented as Add-On. -->

## Specifications

### Word Size
Our word size is 32 bits, which makes a half-word 16 bits, a double-word 64 bits, and so on.

<!-- ### Memory Paging
We will implement a multi-level page table that allows us to have a proper virtual memory system, so we can run subprocesses in peace. The structure of our page table will be detailed below. -->

### Memory Caching
We will use a direct-mapped cache. This will keep our cache implementation simple, although with decreased performance compared to other cache types.

More specifically, we are using a cache with a block size of <>.

Each entry in our cache will have a 16 bit offset, 8 bit index, and a 8 bit tag. This all fits into our word-size, which is 32 bits. 

The exact structure, in C++ syntax, is as follows:

```C++
struct CacheEntry {
    uint32_t Offset: 16;
    uint32_t Index: 8;
    uint32_t Tag: 8;
};
```

A detailed variant of the cache entry layout is as follows:
![Cache Table](diag/cache.png)

### Data Types
The Garand architecture officially has support for just one data type, that being integers (both signed and unsigned formats).

Our architecture defines two types of integers:
-   A 32 bit unsigned integer, with the range [0, $2^{32}-1$].
-   A 32 bit signed integer. In order to represent the negative range of numbers, our architecture will use the two's complement binary representation to store both signed and unsigned numbers. However our range is less than that of the unsigned case; a mere [$-2^{31}$, $2^{31}-1$], or [$-2147483648$, $2147483647$]. 

All operations which operate on unsigned numbers will support operations on the signed equivalents, but sign extension will be performed on signed integers to maintain proper representation.

### Recommended Implementations of Other Data Types
#### Fixed Point 
In various computation-heavy fields, such as computer graphics, precision in calculations is fundamental; from raytracing, to interpolation, and more. 

As such, it is fundamental that, for a graphics-based architecture, we have some sort of mechanism for precise calculations. Using the integer data type, one can trivially implement operations for fixed point numbers. 

Our recommended implementation is rather straightforward. If one wants to maintain $n$ bits worth of precision in a fixed point operation, then:
- Reserve the lower $n$ bits of a word to store the fractional element of the number, which is the number succeeding the decimal point. 
- Reserve at minimum $32 - n - 1$ bits for the integer element of the number, which is the number preceding the decimal point. We recommend a minimum of $32 - n - 1$ only if you care about signed fixed point numbers. If you do not care for this, then simply reserve $32 - n$ bits instead.
- From there, whenever you do an operation on a fixed point number involving an immediate, you need to add `1 << n` to the immediate before performing the operation. 

An example implementation of fixed-point can be seen below, for $n = 23$. This implementation reserves 23 bits worth of precision for the fractional part, 8 bits for the integer part, and 1 bit for the sign.

![Fixed Point Spec](diag/fx.png)

### Registers
Our architecture is designed to support 36 total registers. The breakdown is as follows:

-   16 General-Purpose Registers (Assembler Syntax `R##`).
-   16 IO-specific registers (Assembler Syntax: `I##`)
-   4 special purpose registers:
    - Condition Flag (Assembler Syntax: `CND`) - Used to store and procure the results of carry, overflow, or other conditional operations.
    - Program Counter (Assembler Syntax: `PC`) - Points to the currently executed instruction.
    - Link Register (Assembler Syntax: `LR`) - Commonly stores an address to a function (used in the caller-callee specification).
    - Stack Pointer (Assembler Syntax: `SP`) - Points to a currently used space on the stack.

Each of these registers uses our word-size, that being 32 bits, as underlying data type.

#### Encoding

| Name  | Storage Size (bits) | Internal Index (Encoding) | Description                    |
| ----- | ------------------- | ------------------------- | ------------------------------ |
| `R##` | 32                  | 0-15                      | General-Purpose Registers 0-15 |
| `I##` | 32                  | 16-31                     | IO-specific Registers 0-15     |
| `SP`  | 32                  | 32                        | Stack Pointer                  |
| `PC`  | 32                  | 33                        | Program Counter                |
| `LR`  | 32                  | 34                        | Link Register                  |

### Fetching Model
As described earlier, our word size is 32 bits. To fully take advantage of this, we will use the `multiple words per instruction` fetch model. This will give us optimal code size while performing relatively efficiently. The only way to fetch data to/from memory is Load and Store.
The corresponding instructions for the two operations are: `MREAD`, `MWRITE`.

### Memory Architecture
We want to keep our instruction and data memory together; as such, we will use the _Princeton_ architecture.

### Addressing Modes

Our architecture supports 2 types of address modes:
-   `Register Indirect`: Register value will be read and used as address for the next execution of the processor. An implementation of this can be seen in the implementation of `BRUH.CC`.
-   `PC + Immediate Offset (or, PC Relative)`: Address for the next execution
of the processor will be calculated using Program Counter (current address) plus
specified offset immediate. This is the recommended method for subroutine branching. An implementation of this can be seen in the implementation of `B.CC`.

### Instructions

#### Encoding
All instructions are encoded with a uniform word-size, and follow one of the following base encoding.
#### General Encoding
![Instruction Encoding](diag/ins_encoding.png)
#### 3 Register Encoding (3R)
![3 Register Instruction Encoding](diag/ins_enc_reg3.png)
#### 2 Register Encoding (3R)
![2 Register Instruction Encoding](diag/ins_enc_reg2.png)
#### 2 Register 1 Immediate Encoding (2R1I)
![2 Register 1 Immediate Encoding](diag/ins_enc_reg2_imm1.png)
#### 1 Register 1 Immediate Encoding (1R1I)
![2 Register 1 Immediate Encoding](diag/ins_enc_reg1_imm1.png)
#### 1 Register Encoding (1R)
![1 Register Encoding](diag/ins_enc_reg1.png)
#### 1 Immediate Encoding (1I)
![1 Immediate Encoding](diag/ins_enc_imm1.png)

#### Operation Codes
A list of the operation codes can be found here. The table is subject to change as instructions are added or removed.

| Opcode   | Variant/Condition | Operation | Encoding |
| -------- | ----------------- | --------- | -------- |
| `000000` | `0000`            | MREAD     | 3R |
| `000000` | `0001`            | MWRITE    | 3R |
| `000000` | -                 | -         | |
| `000000` | -                 | -         | |
| `000010` | *User-Supplied*   | BRUH.CC   | 1R |
| `000011` | *User-Supplied*   | B.CC      | 1I |
| `000100` | `0000`            | ADD       | 3R |
| `000100` | `0001`            | ADDI      | 2R1I |
| `000101` | `0000`            | SUB       | 3R |
| `000101` | `0001`            | SUBI      | 2R1I |
| `000101` | `0010`            | CMP       | 2R |
| `000101` | `0011`            | CMPI      | 1R1I |
| `000110` | `0000`            | MUL       | 3R |
| `000110` | `0001`            | MULI      | 2R1I |
| `000110` | `0100`            | MADD      | 3R |
| `000111` | `0000`            | DIV       | 3R |
| `000111` | `0001`            | DIVI      | 2R1I |
| `001000` | `0000`            | AND       | 2R |
| `001000` | `0001`            | ANDI      | 1R1I |
| `001000` | `0010`            | TEST      | 2R |
| `001001` | `0000`            | NAND      | 2R |
| `001001` | `0001`            | NANDI     | 1R1I |
| `001010` | `0000`            | OR        | 2R |
| `001010` | `0001`            | ORI       | 1R1I |
| `001011` | `0000`            | XOR       | 2R |
| `001011` | `0001`            | XORI      | 1R1I |
| `001100` | `0000`            | LSL       | 3R |
| `001100` | `0001`            | LSLI      | 2R1I |
| `001100` | `0010`            | LSR       | 3R |
| `001100` | `0011`            | LSRI      | 2R1I |
| `001100` | `0100`            | RSR       | 3R |
| `001100` | `0101`            | RSRI      | 2R1I |
| `001111` | `0000`            | NOT       | |


#### Variants & Conditions

##### Variants
Many of the operations in our architecture, specifically arithmetic operations, have variants on them that allow for operations between different parameter types. For example, a common one would be a variant that allows operations on registers with immediate values. As such, we use the `Condition` field to specify the arguments being passed into the mmeonic operation. The architecture then knows to take the parameters and treat them differently. 

**This is not implemented for branches, as they are linked during the binary lowering process.**

##### Conditions
Our architecture supports condition-based execution, based on that of ARM. All four condition flags are stored in the lower four bits of the condition flag register, and (optionally) in the condition field of our instruction encoding. The details of each flag are:

-   `N`: Negative condition flag. Set if the result of the most recent flag-setting instruction is negative.
-   `C`: Carry condition flag. Set if the result of the most recent flag-setting instruction has carry.
-   `Z`: Zero condition flag. Set if the result of the most recent flag-setting instruction is zero.
-   `V`: Overflow condition flag. Set if the result of the most recent flag-setting instruction has overflow.

The layout for each of these flags in our condition fields is as follows:

![Condition Encoding](diag/cond_enc.png)

As the next section details, we combine these flags to implement condition codes.

##### Condition Codes

Condtion codes are used in control-flow instructions to, conditionally, branch or jump to a certain location. They are *optionally* user-specified; if no condition code is specified, it is treated as unconditional.

The instructions that use the condition codes are `BRUH.CC`, and `B.CC`. They will be detailed below.

| Encoding | Mnemonic | Meaning(Integer)               | Flag                  |
| -------- | -------- | ------------------------------ | --------------------- |
| `0000`   | `AL`     | Always                         | Any                   |
| `0001`   | `EQ`     | Equal                          | `Z == 0`              |
| `0010`   | `NE`     | Not equal                      | `Z == 1`              |
| `0011`   | `LO`     | Unsigned less than             | `C == 0`              |
| `0100`   | `HS`     | Unsigned greater than or equal | `C == 1`              |
| `0101`   | `LT`     | Signed less than               | `N != V`              |
| `0110`   | `GE`     | Signed greater than or equal   | `N == V`              |
| `0111`   | `HI`     | Unsigned greater than          | `C == 1 && Z == 0`    |
| `1000`   | `LS`     | Unsigned less than or equal    | `!(C == 1 && Z == 0)` |
| `1001`   | `GT`     | Signed greater than            | `Z == 0 && N == V`    |
| `1010`   | `LE`     | Signed less than or equal      | `!(Z == 0 && N == V)` |
| `1011`   | `VC`     | No overflow                    | `V == 0`              |
| `1100`   | `VS`     | Overflow                       | `V == 1`              |
| `1101`   | `PL`     | Positive or zero               | `N == 0`              |
| `1110`   | `NG`     | Negative                       | `N == 1`              |
| `1111`   |          | *Reserved*                     |                       |

## Instruction Documentation
### Load/Store

#### MREAD
##### Assembler Syntax
`MREAD   RX, RM, RN`

##### Description
Read from memory address and store value in register RX. The memory address
is calculated by adding the base value in register M and the offset value in
register N.

##### C Psuedocode

```c
RX = RM[RN];
```

#### MWRITE

##### Assembler Syntax

`MWRITE   RX, RM, RN`

##### Description
Write to memory address using value stored in register X. The memory address
is calculated b using base value from register M, plus the offset value from
register N.

##### C Psuedocode

```c
RM[RN] = RX;
```


### Control

#### B.CC

##### Assembler Syntax

`B.CC   label`  

##### Description

Change the Program Counter value to a label at a PC-relative offset if the specified `CC` matches the Condition Flag registers, else this is a no-op.
The address is computed by using sum of the current PC, and the immediate in the encoding.
The condition `CC` represents one of the condition codes defined in _Condition Encoding_ section.

##### C Psuedocode

```c
PC = PC + imm;
```

#### BRUH.CC

##### Assembler Syntax

`BRUH.CC RM`  

##### Description

Change the Instruction Pointer to address indicated by register M if the condition
matches the Condition Flag registers, else this is a no-op.
In other words, this instruction performs an absolute jump.
The condition `CC` represents one of the condition codes defined in _Condition Encoding_ section.

##### C Psuedocode

```c
PC = RM;
```

### Arithmetic (Fixed Point, Integer)

#### ADD

##### Assembler Syntax

`ADD    RX, RM, RN`

##### Description

Calculate the two's complement 32-bit sum of two registers M and N.
Store the result in register X.

##### C Psuedocode

```c
RX = RM + RN;
```

#### ADDI

##### Assembler Syntax

`ADDI   RX, RM, #imm`

##### Description

Calculate the two's complement 32-bit sum of register M and immediate.
Store the result in register X.

##### C Psuedocode

```c
RX = RM + imm;
```

#### SUB

##### Assembler Syntax

`SUB    RX, RM, RN`

##### Description

Calculate the two's complement 32-bit difference of two registers M and N.
Store the result in register X.

##### C Psuedocode

```c
RX = RM - RN;
```

#### SUBI

##### Assembler Syntax

`SUBI   RX, RM, #imm`

##### Description

Calculate the two's complement 32-bit difference of register M and immediate.
Store the result in register X.

##### C Psuedocode

```c
RX = RM - imm;
```

#### MUL

##### Assembler Syntax

`MUL    RX, RM, RN`

##### Description

Calculate the two's complement 32-bit product of registers M and N.
Store the result in register X.

##### C Psuedocode

```c
RX = RM * RN;
```

#### MULI

##### Assembler Syntax

`MULI   RX, RM, #imm`

##### Description

Calculate the two's complement 32-bit product of register M and immediate.
Store the result in register X.

##### C Psuedocode

```c
RX = RM * imm;
```

#### DIV

##### Assembler Syntax

`DIV    RX, RM, RN`

##### Description

Calculate the two's complement 32-bit quotient of registers M and N.
Store the result in register X.



##### C Psuedocode

```c
RX = RM / RN;
```

#### DIVI

##### Assembler Syntax

`DIVI   RX, RM, #imm`

##### Description

Calculate the two's complement 32-bit quotient of register M and immediate.
Store the result in register X.



##### C Psuedocode

```c
RX = RM / imm;
```

#### MADD

##### Assembler Syntax

`MADD   RX, RM, RN`

##### Description

Calculate the two's complement 32-bit product of registers M and N.
Add the product with register X, then store the sum in register X.



##### C Psuedocode

```c
RX = RX + (RM * RN);
```

#### CMP

##### Assembler Syntax

`CMP    RM, RN`

##### Description

Compare two registers M and N, then update corresponding flags
in Condition Flag register.

#### CMPI

##### Assembler Syntax

`CMPI   RM, #imm`

##### Description

Compare register M and immediate value and update corresponding flags
in Condition Flag register.

#### TEST

##### Assembler Syntax

`TEST   RM, RN`

##### Description

Calculate bitwise AND on two registers M and N and update corresponding flags
in Condition Flag register.

#### AND

##### Assembler Syntax

`AND    RX, RM, RN`

##### Description

Calculate the bitwise AND of registers M and N.
Store the result in register X.

##### C Psuedocode

```c
RX = RM & RN;
```

#### NAND

##### Assembler Syntax

`NAND   RX, RM, RN`

##### Description

Calculate the bitwise NAND of registers M and N.
Store the result in register X.

##### C Psuedocode

```c
RX = NAND(RM, RN);
```

#### OR

##### Assembler Syntax

`OR     RX, RM, RN`

##### Description

Calculate the bitwise OR of registers M and N.
Store the result in register X.

##### C Psuedocode

```c
RX = RM | RN;
```

#### XOR

##### Assembler Syntax

`XOR    RX, RM, RN`

##### Description

Calculate the bitwise XOR of registers M and N.
Store the result in register X.

##### C Psuedocode

```c
RX = RM ^ RN;
```

#### NOT

##### Assembler Syntax

`NOT    RX, RM`

##### Description

Calculate the bitwise NOT of registers M.
Store the result in register X.

##### C Psuedocode

```c
RX = ~RM;
```

#### ASR

##### Assembler Syntax

`ASR    RX, RM, RN`

##### Description

Arithmetically shift value of Register M right by a variable number of bits.
The amount of shift is from Register N value.
Store the result in register X.

##### C Psuedocode

```c
RX = RM >> RN;
```

#### ASRI

##### Assembler Syntax

`ASR    RX, RM, #imm`

##### Description

Arithmetically shift value of Register M right by a variable number of bits.
The amount of shift is from an immediate value.
Store the result in register X.

##### C Psuedocode

```c
RX = RM >> imm;
```

#### LSR

##### Assembler Syntax

`LSR    RX, RM, RN`

##### Description

Logically shift value of Register M right by a variable number of bits.
The amount of shift is from Register N value.
Store the result in register X.

##### C Psuedocode

```c
RX = RM >> RN;
```

#### LSRI

##### Assembler Syntax

`LSR    RX, RM, #imm`

##### Description

Logically shift value of Register M right by a variable number of bits.
The amount of shift is from an immediate value.
Store the result in register X.

##### C Psuedocode

```c
RX = RM >> imm;
```

#### LSL

##### Assembler Syntax

`LSL    RX, RM, RN`

##### Description

Logically shift value of Register M left by a variable number of bits.
The amount of shift is from Register N value.
Store the result in register X.

##### C Psuedocode

```C
RX = RM << RN;
```

#### LSLI

##### Assembler Syntax

`LSL    RX, RM, #imm`

##### Description

Logically shift value of Register M left by a variable number of bits.
The amount of shift is from an immediate value.
Store the result in register X.

##### C Psuedocode

```C
RX = RM << imm;
```

#### RSR

##### Assembler Syntax

`RSR    RX, RM, RN`

##### Description

Rotationally shift value of Register M right by a variable number of bits.
The amount of shift is from Register N value.
Store the result in register X.

##### C Psuedocode

```c
RX = RSR(RM, RN);
```

#### RSRI

##### Assembler Syntax

`RSR    RX, RM, #imm`

##### Description

Rotationally shift value of Register M right by a variable number of bits.
The amount of shift is from an immediate value.
Store the result in register X.

##### C Psuedocode

```c
RX = RSR(RM, imm);
```

### Special

#### CALL

##### Assembler Syntax

`CALL`

##### Description

Break execution. In the simulator, the processor must pause processing after this point.

## Management Plan

- The simulator for Garand is written in C++, using the C++20 standard.
- The Dear ImGui library is used to provide a Graphical User Interface (GUI) for the simulator.
- All the team members will use Microsoft Visual Studio Code as an IDE to develop the project.
- The source code and this document are version-controlled and hosted on Github.
- Issues and to-do lists are also maintained on Github.
- Our main communication line is Discord. Thrice a week, we will meet at the CICS Makerspace to review project progress and discuss the plan for the next.
- Generally, we aim to share project works equally between members of the team. We plan to divide works as below:
    - ISA definition in C++: Ibrahima Keita
    - Processor implementation: Dung Nguyen, Ibrahima Keita
    - Memory/cache implementation: Sergio Ly, Dung Nguyen
    - Simulator UI: Ibrahima Keita, Sergio Ly
    - Writing Benchmark: Dung Nguyen
    - Benchmarking: Sergio Ly

## Reflection
There were many challenges we faced throughout this project. The most difficult aspect was the fact that we had no prior experience designing an ISA. There were many design choices we made at the beginning that had to be changed or were incomplete due to unforseen issues. A notable example of this is with our isntructions and their encoding. We initially had a single instruction encoding format, but quickly realized that it wasn't sufficient, and thus now have 6 different formats for different instructions. The next challenge we faced was maintaining a large codebase, much of which was duplicated. This was primarily in our instruction implementation, as the instructions were similar yet differed in how to retrieve and handle the data. Our initial implementation had a lot of repeated code, which became unmaintainable. We later changed to having more versatile functions to remove the code duplication. A third major challenge was working with C++ and compiling the project. We used CMake with several dependencies, which was troublesome to get working across all our group members. The compiler would produce several errors, most of which were unhelpful and took weeks to figure out.

Additionally to project specific issues, we noticed the GUI was slowing down the execution of the program significantly. When running our larger benchmarks, we had to find a way to make them fast. Thus, we implemented a frame skipping feature, which would only update the GUI every $n$ clock cycles.

