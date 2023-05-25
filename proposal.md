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

| Name       | Storage Size (bits) | Internal Index (Encoding) | Description                    |
| ---------- | ------------------- | ------------------------- | ------------------------------ |
| `R##`      | 32                  | 0-15                      | General-Purpose Registers 0-15 |
| `I##`      | 32                  | 16-31                     | IO-specific Registers 0-15     |
| `SP`       | 32                  | 32                        | Stack Pointer                  |
| `PC`       | 32                  | 33                        | Program Counter                |
| `LR`       | 32                  | 34                        | Link Register                  |
| `reserved` | 32                  | 35-63                     | `reserved`                     |

### Fetching Model
As described earlier, our word size is 32 bits. We will use the `single word per instruction` fetch model. This will give us optimal code size while performing relatively efficiently. The only way to fetch data to/from memory is Load and Store.
The corresponding instructions for the two operations are: `MREAD`, `MWRITE`.

### Memory Architecture
We want to keep our instruction and data memory together; as such, we will use the _Princeton_ architecture.

### Addressing Modes

Our architecture supports 2 types of address modes:
-   `Register Indirect`: Register value will be read and used as address for the next execution of the processor. An implementation of this can be seen in the implementation of `BRUH.CC`.
-   `Register Direct`: Register value is used for the next execution of the processor. An implementation of this can be seen in the implemenation of `ADD`.
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
| `000000` | `0000`            | MREAD     | 3R       |
| `000000` | `0001`            | MWRITE    | 3R       |
| `000000` | -                 | -         |          |
| `000000` | -                 | -         |          |
| `000010` | *User-Supplied*   | BRUH.CC   | 1R       |
| `000011` | *User-Supplied*   | B.CC      | 1I       |
| `000100` | `0000`            | ADD       | 3R       |
| `000100` | `0001`            | ADDI      | 2R1I     |
| `000101` | `0000`            | SUB       | 3R       |
| `000101` | `0001`            | SUBI      | 2R1I     |
| `000101` | `0010`            | CMP       | 2R       |
| `000101` | `0011`            | CMPI      | 1R1I     |
| `000110` | `0000`            | MUL       | 3R       |
| `000110` | `0001`            | MULI      | 2R1I     |
| `000110` | `0100`            | MADD      | 3R       |
| `000111` | `0000`            | DIV       | 3R       |
| `000111` | `0001`            | DIVI      | 2R1I     |
| `001000` | `0000`            | AND       | 2R       |
| `001000` | `0001`            | ANDI      | 1R1I     |
| `001000` | `0010`            | TEST      | 2R       |
| `001001` | `0000`            | NAND      | 2R       |
| `001001` | `0001`            | NANDI     | 1R1I     |
| `001010` | `0000`            | OR        | 2R       |
| `001010` | `0001`            | ORI       | 1R1I     |
| `001011` | `0000`            | XOR       | 2R       |
| `001011` | `0001`            | XORI      | 1R1I     |
| `001100` | `0000`            | LSL       | 3R       |
| `001100` | `0001`            | LSLI      | 2R1I     |
| `001100` | `0010`            | LSR       | 3R       |
| `001100` | `0011`            | LSRI      | 2R1I     |
| `001100` | `0100`            | RSR       | 3R       |
| `001100` | `0101`            | RSRI      | 2R1I     |
| `001111` | `0000`            | NOT       |          |


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
| `0001`   | `EQ`     | Equal                          | `Z == 1`              |
| `0010`   | `NE`     | Not equal                      | `Z == 0`              |
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
Store the result in register X. Overflow is discarded.

##### C Psuedocode

```c
RX = RM * RN;
```

#### MULI

##### Assembler Syntax

`MULI   RX, RM, #imm`

##### Description

Calculate the two's complement 32-bit product of register M and immediate.
Store the result in register X. Overflow is discarded.

##### C Psuedocode

```c
RX = RM * imm;
```

#### DIV

##### Assembler Syntax

`DIV    RX, RM, RN`

##### Description

Calculate the two's complement 32-bit quotient of registers M and N.
Store the result in register X. Remainder is discarded.



##### C Psuedocode

```c
RX = RM / RN;
```

#### DIVI

##### Assembler Syntax

`DIVI   RX, RM, #imm`

##### Description

Calculate the two's complement 32-bit quotient of register M and immediate.
Store the result in register X. Remainder is discarded.



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
- Eventually, the actual work division is:
  - ISA definition: Ibrahima Keita, Dung Nguyen,  Sergio Ly
  - Processor implementation: Dung Nguyen, Sergio Ly
  - Graphic extension: Ibrahima Keita
  - Memory/cache implementation: Ibrahima Keita, Dung Nguyen
  - Simulator UI: Dung Nguyen
  - Writing Benchmark: Sergio Ly
  - Benchmarking: Sergio Ly

## Benchmarks
We used the two required benchmarks (exchange sort & matrix multiply) to see how our simulator performed based on cache size. We set our cache hit to take 0x5 cycles and miss to take 0x30 cycles.

### Exchange Sort
We benchmarked the exchange sort with a 100 item array. The array was randomly generated. Below is a table of our results. Most notably, having a cache is the most important part of our ISA. Our results show that not having is cache causes a significant slowdown in performance, since there are several more pipeline stalls due to fetching from main memory.
|        | Pipeline & Cache | Cache   | Pipeline  | None      |
| ------ | ---------------- | ------- | --------- | --------- |
| cycles | 308,412          | 515,674 | 2,768,141 | 3,598,903 |

### Matrix multiply
We benchmarked the matrix multiply by multiplying 2 4x4 matrices. Similar to our previous benchmark, the cache is what caused the largest performance reason, though, the pipeline being disabled also made a difference in performance.
|        | Pipeline & Cache | Cache | Pipeline | None   |
| ------ | ---------------- | ----- | -------- | ------ |
| cycles | 5,858            | 9,450 | 53,417   | 63,071 |

### Results
Evidently, the cache is the largest contributor to performance. This make sense as reading from memory is very slow and takes several clock cycles (0x30 in our benchmarks). This simulator shows the need for a cache, one that is ideally large. Additionally, having a pipeline is also important. Although the pipeline doesn't make as much of a difference as the cache, as the program increases in size, the effects of not having a cache become more evident.

## Reflection

There were many challenges we faced throughout this project. The most difficult aspect was the fact that we had no prior experience designing an ISA. There were many design choices we made at the beginning that had to be changed or were incomplete due to unforeseen issues. A notable example of this is with our instructions and their encoding. We initially had a single instruction encoding format, but quickly realized that it wasn't sufficient, and thus now have 6 different formats for different instructions.

The next challenge we faced was maintaining a large codebase, much of which was duplicated. This was primarily in our instruction implementation, as the instructions were similar yet differed in how to retrieve and handle the data. Our initial implementation had a lot of repeated code, which became unmaintainable.
We later knew the existence of meta-programming, but due to its steep learning curve,
it was not fully utilized to remove the code duplication.

A third major challenge was working with C++ and compiling the project. We used CMake with several dependencies, which was troublesome to get working across all our group members. While technically C++ codebase can be cross-platform, the compiler would produce several errors due to several features being implementation-defined.

Additionally to project specific issues, because we call CPU cycle every time
GUI renders, it can slow down the execution of the program significantly. When running our larger benchmarks, we had to find a way to make them fast. Thus, we implemented a frame skipping feature, which would only update the GUI every $n$ clock cycles.

## Manual

The instructions below contain details of building and running the software.
Should there be any questions, create a new issue at
https://github.com/Tensor497/garand-docs.

### Overview

The `garand` project is divided into three repositories:
-   garand: ([github.com/Tensor497/garand](https://github.com/Tensor497/garand)).
    This is the repository for the simulator and disassembler written in C++20.
-   garand-as: ([github.com/Tensor497/garand-as](https://github.com/Tensor497/garand-as)).
    This is the repository for the assembler written in Python 3.11.
-   garand-docs: ([github.com/Tensor497/garand-docs](https://github.com/Tensor497/garand-docs)).
    This is the repository for the documentation. You can find the latest version
    of this documentation here.

It is necessary to clone all the aforementioned repo using `git`.
You may also download ZIP file containing the source code.

### Prerequisites

The following list contains all dependencies you will need to install before
compiling and running this project. The version are extracted from dev machine,
but does not imply the minimum runnable version.
-   garand:
    -   C++ compiler supporting C++20 standard.
        MSVC 16.11, Clang 16, GCC 12 are known to work.
    -   CMake 3.20+
    -   [vcpkg](https://github.com/microsoft/vcpkg).
        You are welcome to try other C++ package manager solutions.
    -   C++ libraries:
        -   fmt 9.1.0
        -   imgui 1.89.3
        -   SDL2 2.6.2
        -   SDL2PP 0.16.1
    -   [Optional] Ninja 1.11.1. This allows parallel build and exporting 
        `compile_commands.json`.
-   garand-as
    -   Python 3.11+
    -   Python libraries:
        -   Pillow 9.4.0
-   garand-docs
    -   pandoc 2.19.2

#### Configuring Prerequisites
##### Garand
First, install `cmake`, `vcpkg`, and a C compiler. 

Afterwards, run the commands below depending on your host system.

#### Windows

```
vcpkg install fmt sdl2 sdl2-image sdl2-mixer imgui[sdl2-binding,sdl2-renderer-binding] sdl2pp
```

#### Linux

```
vcpkg install fmt "sdl2[wayland,x11]" sdl2-image sdl2-mixer "imgui[sdl2-binding,sdl2-renderer-binding]" sdl2pp --recurse
```

__Save the path that it gives you for `vcpkg`. It is important.__

Once you are done, make sure you are in the project root directory, and run the following command to build:
```
cmake . -DCMAKE_TOOLCHAIN_FILE=`<vcpkg_path>`
```

If you are using the CMake extension in Visual Studio Code, you can set the CMake settings as follows:
```JSON
"cmake.configureSettings": {
    "CMAKE_TOOLCHAIN_FILE" : "<vcpkg_path>/scripts/buildsystems/vcpkg.cmake"
},
```

##### Garand Assembler
First, install Python 3.11+. As this is specific to a user's distribution, I will omit the steps for this part.

Then, install the Python Imaging Library (PIL) via `pip` with the following command:
> pip install Pillow

##### Garand Documentation
If you want to generate a formal version of this PDF, you just want to install pandoc.


After following the above, you are now ready to work with Garand!
### Building

#### garand

```sh
cmake -B build -DCMAKE_BUILD_TYPE=Release
cmake --build build
```

A new folder _build_ shall be created. After running commands successfully,
an executable shall exist inside the folder.

#### garand-as

No compilation is needed to build this.

#### garand-docs

```sh
pandoc -i proposal.md -o proposal.pdf
```

### Running

#### garand

Run the executable with no arguments. For example, on Windows, call `garand.exe`,
on Linux, run `./garand`. Be sure to have an existing working window manager.

#### garand-as

```
python garand-as.py <input.gar> <output.bin>
```

#### garand-docs

Use your favorite PDF viewer to view the  generated`proposal.pdf`.

### Using

#### Assembling

First, write Garand assembly and save it into the file. The recommended
file extension is `.gar`, `.garand`. Above this documentation contains
instructions, mnemonics, and their syntax.

To comment, append to the beginning
of the line (except for spaces) with `#`. To create a label, use `def`.
For example, `def begin` will create label begin at the instruction right next
to it (next instruction must be in new line).
The following example is an infinite loop that keeps adding 1 to `R1`.

```asm
# Infinite loop
def begin
    ADDI    R1, R1, #1
    B.AL    begin
```

To import another binary into the output, do `.incbin bin_name Address`,
with `bin_name` be the name of importing binary and `Address` be the import
location.
For example, `.incbin sun.bmp 0x4000` will copy contents in sun.bmp byte-by-byte
into the output at offset 0x4000.

You can also find more example programs in _garand-as/examples/_.

Then, assemble our program using _garand-as_. For example, if our program is
_hello.gar_, run:

```
python garand-as.py hello.gar hello.bin
```

#### Running

Open up _garand_. You should see a window named Control UI with a list of checkboxes.
```
[] Demo Window
[] Memory Demo
[] Instruction Demo
[] Pipeline Demo
[] Disassembler Demo
[] Simulator
```

Click on _Simulator_ to show simulator window. 6 new windows will show up.

First is _Simulator_ window. Here you can find:
-   `Load Offset`: Input offset in memory to load data in. Expect hex value.
-   `Executable`: Input box for path to the executable, which is the output of
    assembler. There should be no quotes wrapping the path.
-   `Load`: Clicking the button, it will read the executable pointed by the
    specified path and write to the memory. An error text will show if loading
    is failed.
-   `PC: 0x...`: Displaying current program counter.
-   `Speculating next: ...`: Displaying instruction next to program counter.
-   `Clock: ... cycle(s)`: Display the total clocks that the CPUs have cycled.
-   `{...}`: Display list of breakpoint address.
-   `Set Breakpoint`: Clicking will add new breakpoint to the list
-   `Remove Breakpoint`: Clicking will remove breakpoint from the list
-   `Breakpoint`: Specify the address of breakpoint to set/remove
-   `IsRunning`: Displaying running status of the CPU. 0 means no. 1 means yes.
-   `Run`: Set `IsRunning` to 1 and let CPU cycles until a breakpoint is hit
    or an error occurs.
-   `Step`: Perform one CPU cycle.
-   `Reset Register`: Set all register values to 0.
-   `Reset Processor`: Reset CPU to initial state.

Second window is `Feature`, here you can find
-   `Turn on Skipping`: Enabling CPU skipping feature. Clicking on the checkbox
    will display an integer input box where you can specify number of skipping
    cycles when running. For example, 1000 means CPU will cycle 1000+1 cycles
    before rendering UI.
-   `Pipeline`: Enabling CPU pipeline. When checked, instruction will be added
    to the CPU pipeline as soon as possible. Else, instruction will only be
    added when the pipeline is empty.
-   `Cache`: Enabling CPU cache. When checked, memory will be loaded via cache
    instead and reduce needed cycle. Else, data will only be copied to and
    from main memory.

Third window is `Pipeline View`, displaying the state of each stage of the
pipeline: `Fetch`, `Decode`, `Execute`, `Write-back`.
At each stage, if not `(empty)`, there are 5 lines of information:
-   `I`: Disassembly of instruction.
    This is provided for reference of what instruction would be executed.
-   `C`: Cycle. Number of cycles that CPU has spent on this instruction at
    that stage.
-   `M: [..., ..., ..., ...]`: Needed Cycle. Number of required cycle that CPU
    need to spend on to complete the instruction at each stage.
-   `P`: Processed. Inform whether the CPU pipeline stage has processed the
    instruction.
-   `D: [..., ...], | ...`: Decode info. Contains raw decoded arguments of the instruction.

Forth and fifth are `Cache View` and `Memory View`. As the names suggest,
they display current memory and cache used by the CPU.
Both has `Options` button and `Range` input box. `Options` allows customizing
the view, such as number of columns, ASCII, etc.  `Range` allows going to
the address within the memory.
Additionally, Cache has `Block` input to select which block to display,
`Tag: ...` to display cache block tag, and `Valid ...` to display cache block
validity.

Sixth window is `Register Table`. This will display all value of all register
in decimal, except for condition register `NZCV`, which is shown as 1 bit for
each flag.

To load program,
input the path of previously assembled program into `Executable`.
Click `Load` to load the binary. Now you should see in the updated `Memory View`
window now contains the raw bytes of executable.

Additionally, you may set breakpoint on the address of where you want to pause the execution. Since program is load into a larger memory space, you might need to
set breakpoint at the end of program to avoid executing unwanted instruction.

To run, click Run in `Simulator` window. You should be able to see instructions
loaded into pipeline in `Pipeline View` window. If Cache is enabled,
you will also see `Cache View` being updated with value from memory.
If the program also write to  emory, you should be able to see `Memory View`
updating in realtime. If your code use graphics, it would be rendered behind the
windows.
