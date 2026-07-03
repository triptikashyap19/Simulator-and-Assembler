# 16-Bit Custom ISA Assembler & Simulator

A comprehensive implementation of an **Assembler** and a **Simulator** for a custom 16-bit Instruction Set Architecture (ISA). This project features support for integer operations, condition flags, branching, memory loads/stores, and a custom 8-bit floating-point implementation.

---

## 📖 Table of Contents
1. [System Specifications](#-system-specifications)
2. [Instruction Formats](#-instruction-formats)
3. [Instruction Set Architecture (ISA) Table](#-instruction-set-architecture-isa-table)
4. [Custom 8-bit Floating Point Representation](#-custom-8-bit-floating-point-representation)
5. [Assembler & Error Handling](#-assembler--error-handling)
6. [Simulator Trace & Output](#-simulator-trace--output)
7. [System Architecture Designer Helper (Q5)](#-system-architecture-designer-helper-q5)
8. [Getting Started & Usage](#-getting-started--usage)

---

## ⚙️ System Specifications

- **Instruction Size**: 16 bits
- **Memory Addressability**: Byte-addressable, supporting up to 256 bytes (8-bit memory addresses).
- **General-Purpose Registers (GPRs)**: 7 registers named `R0` through `R6` (3-bit binary addresses `000` to `110`).
- **Condition Register (`FLAGS`)**: A special 16-bit register (`111`) containing flags that are set by arithmetic, comparison, and logical instructions:
  - **V (Overflow)**: Bit index 3 (value 8) - Set if an arithmetic operation overflows or underflows.
  - **L (Less Than)**: Bit index 2 (value 4) - Set if the first operand is less than the second in a comparison.
  - **G (Greater Than)**: Bit index 1 (value 2) - Set if the first operand is greater than the second in a comparison.
  - **E (Equal)**: Bit index 0 (value 1) - Set if operands are equal in a comparison.

---

## 🛠️ Instruction Formats

Instructions are classified into 6 categories based on their parameter structure, each padded with unused "filler" bits to align to exactly 16 bits:

| Type | Format | Fields | Example |
| :---: | :--- | :--- | :--- |
| **Type A** | `[Opcode:5] [Filler:2] [Reg1:3] [Reg2:3] [Reg3:3]` | 3 Registers | `add R1 R2 R3` |
| **Type B** | `[Opcode:5] [Reg1:3] [Immediate:8]` | Register + Immediate | `mov R1 $24` |
| **Type C** | `[Opcode:5] [Filler:5] [Reg1:3] [Reg2:3]` | Register + Register | `mov R2 FLAGS` |
| **Type D** | `[Opcode:5] [Reg1:3] [Memory_Address:8]` | Register + Memory Addr | `ld R1 var1` |
| **Type E** | `[Opcode:5] [Filler:3] [Memory_Address:8]` | Memory Address (Jumps) | `jmp label1` |
| **Type F** | `[Opcode:5] [Filler:11]` | Halt | `hlt` |

---

## 📊 Instruction Set Architecture (ISA) Table

| Mnemonic | Opcode | Type | Syntax | Description |
| :--- | :---: | :---: | :--- | :--- |
| **add** | `10000` | A | `add Reg1 Reg2 Reg3` | `Reg1 = Reg2 + Reg3`. Sets overflow (`V`) if result > 65535. |
| **sub** | `10001` | A | `sub Reg1 Reg2 Reg3` | `Reg1 = Reg2 - Reg3`. Sets overflow (`V`) if result < 0 (clips to 0). |
| **mov** | `10010` | B | `mov Reg1 $imm` | `Reg1 = Immediate` (Immediate range: `0` to `255`). |
| **mov** | `10011` | C | `mov Reg1 Reg2` | `Reg2 = Reg1` (can read from `FLAGS` register). |
| **ld** | `10100` | D | `ld Reg1 mem_addr` | Load: `Reg1 = Memory[mem_addr]`. |
| **st** | `10101` | D | `st Reg1 mem_addr` | Store: `Memory[mem_addr] = Reg1`. |
| **mul** | `10110` | A | `mul Reg1 Reg2 Reg3` | `Reg1 = Reg2 * Reg3`. Sets overflow (`V`) if result > 65535. |
| **div** | `10111` | C | `div Reg1 Reg2` | Divide: `R0 = Reg1 / Reg2`, `R1 = Reg1 % Reg2`. |
| **rs** | `11000` | B | `rs Reg1 $imm` | Right shift: `Reg1 = Reg1 >> imm`. Sets overflow if underflow. |
| **ls** | `11001` | B | `ls Reg1 $imm` | Left shift: `Reg1 = Reg1 << imm`. Sets overflow if result > 65535. |
| **xor** | `11010` | A | `xor Reg1 Reg2 Reg3` | Bitwise XOR: `Reg1 = Reg2 ^ Reg3`. |
| **or** | `11011` | A | `or Reg1 Reg2 Reg3` | Bitwise OR: `Reg1 = Reg2 \| Reg3`. |
| **and** | `11100` | A | `and Reg1 Reg2 Reg3` | Bitwise AND: `Reg1 = Reg2 & Reg3`. |
| **not** | `11101` | C | `not Reg1 Reg2` | Bitwise NOT: `Reg1 = ~Reg2` (16-bit inversion). |
| **cmp** | `11110` | C | `cmp Reg1 Reg2` | Compare: Updates `FLAGS` register (`E=1` if equal, `G=2` if `Reg1 > Reg2`, `L=4` if `Reg1 < Reg2`). |
| **jmp** | `11111` | E | `jmp mem_addr` | Unconditional jump to `mem_addr`. |
| **jlt** | `01100` | E | `jlt mem_addr` | Jump to `mem_addr` if Less than flag `L` is set (`FLAGS == 4`). |
| **jgt** | `01101` | E | `jgt mem_addr` | Jump to `mem_addr` if Greater than flag `G` is set (`FLAGS == 2`). |
| **je** | `01111` | E | `je mem_addr` | Jump to `mem_addr` if Equal flag `E` is set (`FLAGS == 1`). |
| **hlt** | `01010` | F | `hlt` | Halts simulation execution. |
| **addf** | `00000` | A | `addf Reg1 Reg2 Reg3` | Floating Point Addition: `Reg1 = Reg2 + Reg3`. Sets overflow (`V`) if result > 252.0. |
| **subf** | `00001` | A | `subf Reg1 Reg2 Reg3` | Floating Point Subtraction: `Reg1 = Reg2 - Reg3`. Sets overflow (`V`) if result < 0 (clips to 0). |
| **movf** | `00010` | B | `movf Reg1 $float_imm` | Load Floating Point Immediate: `Reg1 = float_imm` (using 8-bit float representation). |

---

## 🧮 Custom 8-bit Floating Point Representation

Floating-point numbers in the extended assembler (`q3_floatingPointAssembler.py`) and simulator use a custom **8-bit format**:
- **Format**: `[Exponent: 3 bits] [Mantissa: 5 bits]`
- **Encoding Rule**:
  - The number is expressed in binary scientific notation as: `1.mantissa_bits * 2^exponent`.
  - The 3 bits of exponent represent `exponent - 1` (where the exponent ranges from 1 to 8).
  - The 5 bits of mantissa represent the fractional part following the implicit leading `1.`.
  - Numbers less than `1.0` cannot be represented and will throw an error.
  - Overflow limit is `252.0`. Exponents larger than 7 (representing `2^7` scale) will trigger overflow.

---

## 🔍 Assembler & Error Handling

The Assembler reads assembly instructions from `sys.stdin` and outputs 16-bit binary representations to `sys.stdout`. It performs comprehensive validation:
1. **Instruction & Register Validity**: Verifies mnemonics and checks that registers are inside the `R0`-`R6` range (illegal usage of `FLAGS` causes immediate failure).
2. **Placement of Variables**: Variables must be declared using `var <variable_name>` strictly at the very beginning of the program.
3. **Label Checks**: Ensures labels are alphanumeric, not duplicate, and do not clash with variables.
4. **Immediate Constraints**: Integer immediates must fit within 8 bits (`0` to `255`), and floating-point values must be representable within the 8-bit float layout.
5. **Program Size Limit**: Throws an error if code exceeds 256 lines (due to the 8-bit memory constraints).
6. **Halt Check**: Ensures that there is exactly one `hlt` instruction, and that it is the final instruction of the program.

---

## 🖥️ Simulator Trace & Output

The Simulator executes the binary file line-by-line and dumps the CPU state at each cycle to stdout.
- **Each trace line contains**:
  `[Program Counter: 8-bit Binary] [R0: 16-bit Binary] ... [R6: 16-bit Binary] [FLAGS: 16-bit Binary]`
- **Memory Dump**:
  Upon hitting `hlt`, the simulator outputs the complete 256-word memory array (each line representing a 16-bit binary value).
- **Execution Cycles**:
  Execution loops up to a maximum safety threshold of `100,000` cycles to prevent infinite loops.

---

## 📐 System Architecture Designer Helper (Q5)

Located in `SimpleSimulator/Q5.py`, this interactive utility helps system architects design custom ISAs. Given basic parameters, it outputs:
1. Minimum address bits required.
2. Bitwidth allocated to the opcode.
3. Count of filler bits.
4. Maximum addressable registers and instructions.
5. Bit-pin scaling conversions under different addressable units (Bit, Nibble, Byte, Word).

---

## 🚀 Getting Started & Usage

### Running the Assembler
Use the script or execute the Python files directly:
```bash
# Using standard integer assembler
python3 Simple-Assembler/SimpleAssembler.py < input.asm > output.bin

# Using floating point-capable assembler
python3 Simple-Assembler/q3_floatingPointAssembler.py < input.asm > output.bin
```

### Running the Simulator
To run the simulation and trace register/memory states:
```bash
python3 SimpleSimulator/memory.py < output.bin
```

### Pipelines / Full Flow
You can pipe the assembler directly into the simulator to test your programs end-to-end:
```bash
python3 Simple-Assembler/SimpleAssembler.py < input.asm | python3 SimpleSimulator/memory.py
```
