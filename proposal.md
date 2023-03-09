---
title: Garand Architecture
author: Ibrahima Keita, Dung Nguyen, Sergio Ly
geometry: margin=2cm
output: pdf_document
---

<!-- # Garand Architecture -->

## Description
The Garand Architecture is a special-purpose architecture designed with gaming and graphics processing in mind.

<!-- Named Garand Architecture. As a Special Purpose architecture derived from RISC/ARM, its primary support is gaming and graphics processing. It contains low-level graphics support, compatible with openGL. Multiple co-processing is also possible. Vectorization such as Fused Multiplacation-Add is included. On Developer side, extensible interface is documented as Add-On. -->

## Specifications

### Word Size
Our word size is 32 bits, which makes a half word 16 bits, a double word 64 bits, and so on.

<!-- ### Memory Paging
We will implement a multi-level page table that allows us to have a proper virtual memory system, so we can run subprocesses in peace. The structure of our page table will be detailed below. -->

### Memory Caching
We will use a direct mapped cache. This will keep our cache implementation simple, although with decreased performance. Our cache will also use the Modified Shared Invalid (MSI) protocol to ensure cache coherency in our multi-core architecture.

Each entry in our cache will have a 16 bit offset, 8 bit index, 7 bit tag, and 1 dirtybit. The implementation of our table is as follows:
```c++
struct CacheEntry {
    offset: 16;
    index: 8;
    tag: 7
    dirty: 1
};

struct CacheTable {
    CacheEntry entries[32];
};
```

![Cache Table](diag/cache.png)

### Data Types
In terms of data types, we aim to support two distinct numerical types: integer and fixed point.

#### Integer
Our architecture defines two types of integers:
-   A 32 bit unsigned integer, with the range [0, $2^{32}-1$].
-   A 32 bit signed integer. To simulate the negative range of numbers, our instruction set architecture will use two's complement to store distinct numbers. Therefore, our range is [$-2^{31}$, $2^{31}-1$], or [-2147483648, 2147483647]. All operations which operate on unsigned numbers will support operations on the signed equivalents, but sign extension will be performed to maintain proper two's complement support.

#### Fixed Point 
In graphics, precision in calculations is fundamental; from raytracing, to interpolation, etc. As such, it is fundamental that, for a graphics-based architecture, we have some sort of mechanism for precise calculations. Therefore, we will be implementing fixed point precision.

For our implementation of fixed point, we will reserve the lower 23 bits of a word to store our _fractional_, which is the number that comes after the decimal point. Our _exponent_, which is 8 bits, will immediately follow the _fractional_ in bit order, taking bits 23 through 30 inclusive. Finally, the most significant bit (which is bit 31) will act as a signage bit, allowing us to have positive and negative fixed-point values.

A detailed diagram of this can be seen below.

![Fixed Point Spec](diag/fx.png)

As these numbers require special treatment, we will implement instructions specifically for them.

### Registers
Our architecture is designed to support 36 total registers. The breakdown is as follows:

-   16 General-Purpose Registers (assembler syntax `R##`).
-   16 IO-specific registers (assembler syntax: `I##`)
-   4 special purpose registers:
    - Condition Flag (assembler syntax: `CND`) - Used to store and procure the results of carry, overflow, or other conditional operations.
    - Program Counter (assembler syntax: `PC`) - Points to the currently executed instruction.
    - Link Register (assembler syntax: `LR`) - Commonly stores an address to a function (used in the caller-callee specification).
    - Stack Pointer (assembler syntax: `SP`) - Points to a currently used space on the stack.

Each of these registers uses a single word (32 bits) as its underlying data type.

#### Encoding

| Name  | Size (bits) | Encoding | Description                    |
| ----- | ----------- | -------- | ------------------------------ |
| `R##` | 32          | 0-15     | General-Purpose Registers 0-15 |
| `I##` | 32          | 16-31    | IO-specific Registers 0-15     |
| `SP`  | 32          | 32       | Stack Pointer                  |
| `PC`  | 32          | 33       | Program Counter                |
| `LR`  | 32          | 34       | Link Register                  |

#### Binding & Locking
As we are writing a multi-core architecture, we are very likely to run into race conditions when writing to/reading from registers. A special feature in each core is thus needed.

As such, at the hardware level, we will implement an atomic locking mechanism that activates when a core writes to a register. The way this works is simple: the first core that writes to a register will have its operations carried out, while the other cores will have to wait until the lock is released. We will keep track of this using a table protected by atomic locks.

Furthermore, we are also going to introduce the concept of register binding for IO registers. The motivation behind this is to create a hassle-free experience for a developer that is interfacing with hardware devices. This will be provided in the form of the `BIND` instruction, which will take an IO-specific register, store the memory address of an IO device (which would likely be mapped to an index) in one of the IO registers, and protect the value in the IO-register from being overwritten. That way, we do not need to do static memory writes, and we can directly write to a register bound to IO. When we are ready to clean up the system, we just use the `UNBIND` instruction, which removes our IO register from the binding table.


### Fetching Model
As described earlier, our word size is 32 bits. To fully take advantage of this, we will use the `multiple words per instruction` fetch model. This will give us optimal code size while performing relatively efficiently. The only way to fetch data to/from memory is Load and Store.
The corresponding instructions for the two operations are: `MREAD`, `MWRITE`.

### Memory Architecture
We want to keep our instruction and data memory together; as such, we will use the _Princeton_ architecture.

### Addressing Modes

Our architecture supports 2 types of address modes:

-   Register: Register value will be read and used as address for the next execution
of the processor. For instruction details, see `BRUH.CC`
-   PC + Immediate Offset (PC Relative): Address for the next execution
of the processor will be calculated using Program Counter (current address) plus
specified offset immediate. This is the recommended method for subroutine branching.
For instruction details, see `B.CC`

### Instructions

#### Encoding
When choosing the encoding for this architecture, we wanted to go for an approach that yields uniform instruction sizes, while being able to perform as many operations as possible. Therefore, all instructions will follow the base encoding shown below.

![Instruction Encoding](diag/ins_encoding.png)


#### Operation Codes
A list of the operation codes can be found here. The table is subjected to change in the future.

| Opcode   | Condition | Operation |
| -------- | --------- | --------- |
| `000000` | `0000`    | MREAD     |
| `000000` | `0001`    | MWRITE    |
| `000000` | `0010`    | BIND      |
| `000000` | `0011`    | UNBIND    |
| `000010` | -         | BRUH.CC   |
| `000011` | -         | B.CC      |
| `000100` | `0000`    | ADD       |
| `000100` | `0001`    | ADDI      |
| `000100` | `1000`    | FX_ADD    |
| `000100` | `1001`    | FX_ADDI   |
| `000101` | `0000`    | SUB       |
| `000101` | `0001`    | SUBI      |
| `000101` | `0010`    | CMP       |
| `000101` | `0011`    | CMPI      |
| `000101` | `1000`    | FX_SUB    |
| `000101` | `1001`    | FX_SUBI   |
| `000110` | `0000`    | MUL       |
| `000110` | `0001`    | MULI      |
| `000110` | `0100`    | MADD      |
| `000110` | `1000`    | FX_MUL    |
| `000110` | `1001`    | FX_MULI   |
| `000110` | `1100`    | FX_MADD   |
| `000111` | `0000`    | DIV       |
| `000111` | `0001`    | DIVI      |
| `000111` | `1000`    | FX_DIV    |
| `000111` | `1001`    | FX_DIVI   |
| `001000` | `0000`    | AND       |
| `001000` | `0001`    | ANDI      |
| `001000` | `0010`    | TEST      |
| `001001` | `0000`    | NAND      |
| `001001` | `0001`    | NANDI     |
| `001010` | `0000`    | OR        |
| `001010` | `0001`    | ORI       |
| `001011` | `0000`    | XOR       |
| `001011` | `0001`    | XORI      |
| `001100` | `0000`    | LSL       |
| `001100` | `0001`    | LSLI      |
| `001100` | `0010`    | LSR       |
| `001100` | `0011`    | LSRI      |
| `001100` | `0100`    | RSR       |
| `001100` | `0101`    | RSRI      |
| `001111` | `0000`    | NOT       |


#### Condition encoding

##### Condition flags

Our architecture supports condition flags, based on the specifications from ARM. All four condition flags are stored in the lower four bits of the condition flag register, and (optionally) in the condition field of our instruction encoding. The details of each flag are:

-   `N`: Negative condition flag. Set if the result of the most recent flag-setting instruction is negative.
-   `C`: Carry condition flag. Set if the result of the most recent flag-setting instruction has carry.
-   `Z`: Zero condition flag. Set if the result of the most recent flag-setting instruction is zero.
-   `V`: Overflow condition flag. Set if the result of the most recent flag-setting instruction has overflow.

The layout for each of these flags in our condition fields is as follows:

![Condition Encoding](diag/cond_enc.png)

However, as you will see in the next section, we will be combining these flags to implement condition codes.

##### Condition codes

Condtion codes are used in control instructions to perform conditional branch or jump.
Current instructions use condition codes are: `BRUH.CC`, `B.CC`.

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

##### Assembler syntax

`MREAD   RX, RM, RN`

##### Description

Read from memory address and store value in register X. The memory address
is calculated by base value from register M, plus the offset value from
register N.

##### Psuedo C-code

```c
RX = RM[RN];
```

#### MWRITE

##### Assembler syntax

`MWRITE   RX, RM, RN`

##### Description
Write to memory address using value stored in register X. The memory address
is calculated b using base value from register M, plus the offset value from
register N.

##### Psuedo C-code

```c
RM[RN] = RX;
```

#### BIND

##### Assembler syntax

`BIND   IX, #imm`

##### Description

Bind a IO-specific register to a specific IO device memory.
IO device at address `imm` will be write-protected.
See _Binding & Locking_ section for more details.


#### UNBIND

##### Assembler syntax

`UNBIND IX`

##### Description

Unbind a IO-specific register which was used for be write-protecting IO device memory.
See _Binding & Locking_ section for more details.

### Control

#### B.CC

##### Assembler syntax

`B.CC   label`  

##### Description

Change the Program Counter value to a label at a PC-relative offset if the condition
matches the Condition Flag registers, else this is a no-op.
The address is computed by using sum of PC and immdiate in encoding.
The condition `CC` represents one of the condition codes defined in _Condition Encoding_ section.

##### Psuedo C-code

```c
PC = PC + imm;
```

#### BRUH.CC

##### Assembler syntax

`BRUH.CC RM`  

##### Description

Change the Instruction Pointer to address indicated by register M if the condition
matches the Condition Flag registers, else this is a no-op.
In other words, this instruction performs an absolute jump.
The condition `CC` represents one of the condition codes defined in _Condition Encoding_ section.

##### Psuedo C-code

```c
PC = RM;
```

### Arithmetic (Fixed Point, Integer)

#### ADD

##### Assembler syntax

`[FX_]ADD    RX, RM, RN`

##### Description

For integer version: Calculate the two's complement 32-bit sum of two registers M and N.
Store the result in register X.

Fixed-point version with `FX_` prefix will do similar calculation, but compute on and return fixed point number instead. 

##### Psuedo C-code

```c
RX = RM + RN;
```

#### ADDI

##### Assembler syntax

`[FX_]ADDI   RX, RM, #imm`

##### Description

For integer version: Calculate the two's complement 32-bit sum of register M and immediate.
Store the result in register X.

Fixed-point version with `FX_` prefix will do similar calculation, but compute on and return fixed point number instead. 

##### Psuedo C-code

```c
RX = RM + imm;
```

#### SUB

##### Assembler syntax

`[FX_]SUB    RX, RM, RN`

##### Description

For integer version: Calculate the two's complement 32-bit difference of two registers M and N.
Store the result in register X.

Fixed-point version with `FX_` prefix will do similar calculation, but compute on and return fixed point number instead. 

##### Psuedo C-code

```c
RX = RM - RN;
```

#### SUBI

##### Assembler syntax

`[FX_]SUBI   RX, RM, #imm`

##### Description

For integer version: Calculate the two's complement 32-bit difference of register M and immediate.
Store the result in register X.

Fixed-point version with `FX_` prefix will do similar calculation, but compute on and return fixed point number instead. 

##### Psuedo C-code

```c
RX = RM - imm;
```

#### MUL

##### Assembler syntax

`[FX_]MUL    RX, RM, RN`

##### Description

For integer version: Calculate the two's complement 32-bit product of registers M and N.
Store the result in register X.

Fixed-point version with `FX_` prefix will do similar calculation, but compute on and return fixed point number instead. 

##### Psuedo C-code

```c
RX = RM * RN;
```

#### MULI

##### Assembler syntax

`[FX_]MULI   RX, RM, #imm`

##### Description

For integer version: Calculate the two's complement 32-bit product of register M and immediate.
Store the result in register X.

Fixed-point version with `FX_` prefix will do similar calculation, but compute on and return fixed point number instead. 

##### Psuedo C-code

```c
RX = RM * imm;
```

#### DIV

##### Assembler syntax

`[FX_]DIV    RX, RM, RN`

##### Description

For integer version: Calculate the two's complement 32-bit quotient of registers M and N.
Store the result in register X.

Fixed-point version with `FX_` prefix will do similar calculation, but compute on and return fixed point number instead. 

##### Psuedo C-code

```c
RX = RM / RN;
```

#### DIVI

##### Assembler syntax

`[FX_]DIVI   RX, RM, #imm`

##### Description

For integer version: Calculate the two's complement 32-bit quotient of register M and immediate.
Store the result in register X.

Fixed-point version with `FX_` prefix will do similar calculation, but compute on and return fixed point number instead. 

##### Psuedo C-code

```c
RX = RM / imm;
```

#### MADD

##### Assembler syntax

`[FX_]MADD   RX, RM, RN`

##### Description

For integer version: Calculate the two's complement 32-bit product of registers M and N.
Add the product with register X, then store the sum in register X.

Fixed-point version with `FX_` prefix will do similar calculation, but compute on and return fixed point number instead. 

##### Psuedo C-code

```c
RX = RX + (RM * RN);
```

#### CMP

##### Assembler syntax

`CMP    RM, RN`

##### Description

Compare two registers M and N, then update corresponding flags
in Condition Flag register.

#### CMPI

##### Assembler syntax

`CMPI   RM, #imm`

##### Description

Compare register M and immediate value and update corresponding flags
in Condition Flag register.

#### TEST

##### Assembler syntax

`TEST   RM, RN`

##### Description

Calculate bitwise AND on two registers M and N and update corresponding flags
in Condition Flag register.

#### AND

##### Assembler syntax

`AND    RX, RM, RN`

##### Description

Calculate the bitwise AND of registers M and N.
Store the result in register X.

##### Psuedo C-code

```c
RX = RM & RN;
```

#### NAND

##### Assembler syntax

`NAND   RX, RM, RN`

##### Description

Calculate the bitwise NAND of registers M and N.
Store the result in register X.

##### Psuedo C-code

```c
RX = NAND(RM, RN);
```

#### OR

##### Assembler syntax

`OR     RX, RM, RN`

##### Description

Calculate the bitwise OR of registers M and N.
Store the result in register X.

##### Psuedo C-code

```c
RX = RM | RN;
```

#### XOR

##### Assembler syntax

`XOR    RX, RM, RN`

##### Description

Calculate the bitwise XOR of registers M and N.
Store the result in register X.

##### Psuedo C-code

```c
RX = RM ^ RN;
```

#### NOT

##### Assembler syntax

`NOT    RX, RM`

##### Description

Calculate the bitwise NOT of registers M.
Store the result in register X.

##### Psuedo C-code

```c
RX = ~RM;
```

#### ASR

##### Assembler syntax

`ASR    RX, RM, RN`

##### Description

Arithmetically shift value of Register M right by a variable number of bits.
The amount of shift is from Register N value.
Store the result in register X.

##### Psuedo C-code

```c
RX = RM >> RN;
```

#### LSR

##### Assembler syntax

`LSR    RX, RM, RN`

##### Description

Logically shift value of Register M right by a variable number of bits.
The amount of shift is from Register N value.
Store the result in register X.

##### Psuedo C-code

```c
RX = RM >> RN;
```

#### LSL

##### Assembler syntax

`LSL    RX, RM, RN`

##### Description

Logically shift value of Register M left by a variable number of bits.
The amount of shift is from Register N value.
Store the result in register X.

##### Psuedo C-code

```c
RX = RM << RN;
```

#### RSR

##### Assembler syntax

`RSR    RX, RM, RN`

##### Description

Rotationally shift value of Register M right by a variable number of bits.
The amount of shift is from Register N value.
Store the result in register X.

##### Psuedo C-code

```c
RX = RSR(RM, RN);
```


### Special

#### BREAK

##### Assembler syntax

`BREAK`

##### Description

Break execution. In the simulator, the processor must pause processing after this point.

#### HALT

##### Assembler syntax

`HALT`

##### Description

Stop all execution. In the simulator, the processor must stop processing at this point and exit the program.

#### NOP

##### Assembler syntax

`NOP`

##### Description

No-operation. The processor will not perform any operation on this instruction
other than increasing PC.

## Management Plan

- The simulator for Garand is written in C++, using the C++20 standard.
- The Dear ImGui library is used to provide a Graphical User Interface (GUI) for the simulator.
- All the team members will use Microsoft Visual Studio Code as an IDE to develop the project.
- The source code and this document are version-controlled and hosted on Github.
- Issues and to-do lists are also maintained on Github.
- Our main communication line is Discord. Thrice a week, we will meet in CICS Makerspace to review project progress and discuss next week plan.
- Generally, we aim to share project works equally between members of the team. We plan to divide works as below:
    - ISA definition in C++: Ibrahima Keita
    - Processor implementation: Dung Nguyen, Ibrahima Keita
    - Memory/cache implementation: Sergio Ly, Dung Nguyen
    - Simulator UI: Ibrahima Keita, Sergio Ly
    - Writing Benchmark: Dung Nguyen
    - Benchmarking: Sergio Ly
