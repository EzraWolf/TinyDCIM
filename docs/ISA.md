
# TinyMOA ISA Documentation

## Overview

**TinyMOA** is a custom RISC-V processor designed for **Compute-in-Memory (CIM)** operations using memristor crossbar arrays. It extends the minimal **RV32EC** instruction set with custom CIM instructions for efficient neural network inference.

### What You're Getting

- **Base ISA**: RV32EC (32-bit RISC-V with Embedded extensions + Compressed instructions)
- **Custom Extensions**: 
  - CIM control instructions (weight loading, matrix-vector multiplication)
  - 32×16 multiply instruction
- **Design Philosophy**: Simple blocking/stalling architecture - CPU waits for CIM operations to complete

### Key Terms for Beginners

- **ISA** = Instruction Set Architecture (the "language" the CPU speaks)
- **RISC-V** = Open-source instruction set (unlike proprietary x86 or ARM)
- **RV32EC** = 32-bit RISC-V, Embedded (16 registers instead of 32), Compressed (16-bit instructions)
- **Compressed Instructions** = 16-bit versions of common 32-bit instructions to save code space
- **CIM** = Compute-in-Memory, performs math operations directly in memory using memristor arrays
- **MVM** = Matrix-Vector Multiplication (core operation for neural networks)

---

## CIM Hardware Architecture

### Corelet Array Structure

```
4×4 Corelet Array
┌─────┬─────┬─────┬─────┐
│ C00 │ C01 │ C02 │ C03 │  Each corelet contains:
├─────┼─────┼─────┼─────┤  - 4×4 memristor array (16 cells)
│ C10 │ C11 │ C12 │ C13 │  - 1 neuron (SAR ADC)
├─────┼─────┼─────┼─────┤
│ C20 │ C21 │ C22 │ C23 │  Total: 16×16 = 256 memristors
├─────┼─────┼─────┼─────┤        = 256 multiply-accumulates (MACs)
│ C30 │ C31 │ C32 │ C33 │           per inference cycle
└─────┴─────┴─────┴─────┘
```

### Precision Modes (2-bit encoding)

| Code | Precision | SAR Cycles | Use Case |
|------|-----------|------------|----------|
| `00` | 1-bit | 1 | Binary neural networks |
| `01` | 2-bit | 2 | Low-precision quantized networks |
| `10` | 4-bit | 4 | Standard quantized networks |
| `11` | 8-bit | 8 | High-precision inference |

---

## CIM Operations Pseudocode

### MVM Modes (set via FS instruction)

The **FS (Function State)** register controls which type of matrix-vector multiplication to perform:

1. **Forward MVM**: Input on bitlines (BL) → output on sourcelines (SL)
   - Used for feedforward neural network layers
2. **Backward MVM**: Input on SL → output on BL
   - Used for backpropagation or bidirectional networks  
3. **Recurrent MVM**: Input on BL → output on BL
   - Used for recurrent neural networks (RNNs)

### Weight Loading

```c
LOAD_WEIGHTS(weight_matrix[N][N], precision):
    for each row r in [0..N-1]:
        RS  <- one_hot(r)               // Row Select (activate one row)
        WD  <- weight_matrix[r][0..N-1] // Write Data (all N columns for this row)
        WDS <- N'b111...                // Write Data Strobe (enable all columns)
        FS  <- {WRITE, precision}       // Function State: write mode + precision
        DoA                             // Do Action - CPU blocks until write completes
```

**What happens:** 
- Each row is programmed sequentially
- All 4 columns per row are written simultaneously
- CPU stalls (waits) until programming completes

### Matrix-Vector Multiplication (Inference)

```c
MVM(input_vec[N], precision {1, 2, 4, 8}) -> result[N]:
    SAM <- {result_reg, precision}      // SAR ADC Mode: output register + precision
    acc[0..N-1] <- 0                    // Clear accumulators
    
    // Bit-serial computation (process input bit-by-bit for high precision)
    for b in [0 .. precision-1]:
        // Apply b-th bit of all inputs simultaneously
        RS <- {input_vec[N-1][b],
               input_vec[N-2][b],
               ...,
               input_vec[0][b]}         // Row Select: bit b of all inputs
               
        FS <- {FORWARD_MVM, READ}       // Function State: forward mode, read
        DoA                             // Fire all N rows simultaneously
        DoS                             // Do Sample - latch sample-and-hold

        // Read out results column-by-column
        for c in [0..N-1]:
            CS <- c                     // Column Select
            DoR                         // Do Read - ADC converts, result in result_reg
            acc[c] += (result_reg << b) // CPU shift-adds partial product
    
    return acc[0..N-1]                  // Return N accumulated results
```

**What happens:**
- For 4-bit precision: 4 cycles of bit-serial processing
- Each cycle: all 4 rows fire simultaneously, read out 4 columns sequentially
- CPU accumulates partial products with appropriate bit shifts
- Result: 4-element output vector from 4×4 matrix × 4-element input

---

## Compressed Instruction Set (16-bit)

### Why Compressed Instructions?

Standard RISC-V instructions are **32 bits (4 bytes)** each. **Compressed instructions** squeeze common operations into **16 bits (2 bytes)**, achieving:
- **2× better code density** (program fits in half the memory)
- Faster instruction fetch (fetch 2 instructions at once)
- Lower power consumption

### Understanding the Tables

**Notation Guide:**
- `rd'`, `rs1'`, `rs2'` = **Compressed registers** (only x8-x15, saves 1 bit)
- `rd`, `rs1`, `rs2` = **Full registers** (x0-x15, all 16 registers)
- `imm`, `uimm` = Immediate values (constant numbers embedded in instruction)
- `nzimm`, `nzuimm` = **Non-zero** immediates (imm=0 is reserved/has special meaning)
- `[15:13]` = **Bit range** in the instruction (bits 15 down to 13)
- `RES` = Reserved (will cause error if imm=0)
- `HINT` = No-op for this implementation, reserved for future use
- `NSE` = Non-standard encoding (implementation-specific)

**Why some things are "NOT IMPLEMENTED":**
- **FP instructions** (C.FLD, C.FSD, etc.): Floating-point unit would consume too much area
- **RV64/RV128**: We only implement 32-bit (RV32), not 64-bit or 128-bit variants

### Quadrants

Compressed instructions use bits `[1:0]` to determine **instruction length**:
- `00`, `01`, `10` = **16-bit compressed** (Quadrants 0, 1, 2)
- `11` = **32-bit standard** (Quadrant 3 - not actually a quadrant, just means "use full 32-bit instruction")

---

## Quadrant 0 (`instr[1:0] = 2'b00`) - Memory Operations

**Primary use:** Stack-relative operations and memory access with compressed registers

| [15:13] | [12:5] | [4:2] | [1:0] | Instruction | Description |
|---------|--------|-------|-------|-------------|-------------|
| `000` | `0` | `0` | `00` | **Illegal** | Reserved - causes trap |
| `000` | `nzuimm[5:4\|9:6\|2\|3]` | `rd'` | `00` | **C.ADDI4SPN** | `rd' ← sp + uimm×4` (stack-relative address) |
| `001` | `uimm[5:3], rs1', uimm[7:6]` | `rd'` | `00` | ~~C.FLD~~ | NOT IMPLEMENTED (no floating-point) |
| `010` | `uimm[5:3], rs1', uimm[2\|6]` | `rd'` | `00` | **C.LW** | `rd' ← mem32[rs1' + uimm×4]` (load word) |
| `011` | `uimm[5:3], rs1', uimm[2\|6]` | `rd'` | `00` | ~~C.FLW~~ | NOT IMPLEMENTED (no floating-point) |
| `100` | `-` | `-` | `00` | **Reserved** | Available for custom instructions |
| `101` | `uimm[5:3], rs1', uimm[7:6]` | `rs2'` | `00` | ~~C.FSD~~ | NOT IMPLEMENTED (no floating-point) |
| `110` | `uimm[5:3], rs1', uimm[2\|6]` | `rs2'` | `00` | **C.SW** | `mem32[rs1' + uimm×4] ← rs2'` (store word) |

**Notes:**
- All memory operations use **word-aligned** addresses (offset multiplied by 4)
- Compressed registers (`rd'`, `rs1'`, `rs2'`) only access **x8-x15**
- Floating-point slots (`001`, `011`, `101`) freed up - **4 encodings available for custom CIM instructions**





## Quadrant 1 (`instr[1:0] = 2'b01`) - ALU and Control Flow

**Primary use:** Immediate arithmetic, register operations, and jumps

### C.NOP and C.ADDI

| [15:13] | [12] | [11:7] | [6:2] | [1:0] | Instruction | Description |
|---------|------|--------|-------|-------|-------------|-------------|
| `000` | `nzimm[5]` | `0` | `nzimm[4:0]` | `01` | **C.NOP** | No operation (HINT if nzimm≠0) |
| `000` | `nzimm[5]` | `rd≠0` | `nzimm[4:0]` | `01` | **C.ADDI** | `rd ← rd + sext(nzimm)` (HINT if nzimm=0) |

**Note:** C.NOP is encoded as `C.ADDI x0, nzimm` - writing to x0 has no effect

### C.JAL (Jump and Link)

| [15:13] | [12:2] | [1:0] | Instruction | Description |
|---------|--------|-------|-------------|-------------|
| `001` | `imm[11\|4\|9:8\|10\|6\|7\|3:1\|5]` | `01` | **C.JAL** | `x1 ← pc+2; pc ← pc + sext(imm)` (RV32 only) |

**Note:** Saves return address to **x1 (ra)** and jumps to ±2KiB target

### C.LI (Load Immediate)

| [15:13] | [12] | [11:7] | [6:2] | [1:0] | Instruction | Description |
|---------|------|--------|-------|-------|-------------|-------------|
| `010` | `imm[5]` | `rd≠0` | `imm[4:0]` | `01` | **C.LI** | `rd ← sext(imm)` (load -32 to +31, HINT if rd=0) |

### C.ADDI16SP and C.LUI

| [15:13] | [12] | [11:7] | [6:2] | [1:0] | Instruction | Description |
|---------|------|--------|-------|-------|-------------|-------------|
| `011` | `nzimm[9]` | `2` | `nzimm[4\|6\|8:7\|5]` | `01` | **C.ADDI16SP** | `sp ← sp + sext(nzimm×16)` (RES if nzimm=0) |
| `011` | `nzimm[17]` | `rd≠{0,2}` | `nzimm[16:12]` | `01` | **C.LUI** | `rd ← sext(nzimm) << 12` (RES if nzimm=0, HINT if rd=0) |

**Notes:**
- C.ADDI16SP adjusts stack in **16-byte multiples** for struct alignment
- C.LUI loads upper 20 bits (values 0x1000-0x1F000 or 0xFFFF1000-0xFFFFF000)

### ALU Operations (100 Group)

| [15:13] | [12] | [11:10] | [9:7] | [6:5] | [4:2] | [1:0] | Instruction | Description |
|---------|------|---------|-------|-------|-------|-------|-------------|-------------|
| `100` | `nzuimm[5]` | `00` | `rd'` | `nzuimm[4:0]` | `01` | **C.SRLI** | `rd' ← rd' >> uimm` (RV32 NSE if nzuimm[5]=1) |
| `100` | `nzuimm[5]` | `01` | `rd'` | `nzuimm[4:0]` | `01` | **C.SRAI** | `rd' ← rd' >>> uimm` (RV32 NSE if nzuimm[5]=1) |
| `100` | `imm[5]` | `10` | `rd'` | `imm[4:0]` | `01` | **C.ANDI** | `rd' ← rd' & sext(imm)` (bitwise AND mask) |
| `100` | `0` | `11` | `rd'` | `00` | `rs2'` | `01` | **C.SUB** | `rd' ← rd' - rs2'` (subtract) |
| `100` | `0` | `11` | `rd'` | `01` | `rs2'` | `01` | **C.XOR** | `rd' ← rd' ^ rs2'` (exclusive OR) |
| `100` | `0` | `11` | `rd'` | `10` | `rs2'` | `01` | **C.OR** | `rd' ← rd' \| rs2'` (bitwise OR) |
| `100` | `0` | `11` | `rd'` | `11` | `rs2'` | `01` | **C.AND** | `rd' ← rd' & rs2'` (bitwise AND) |
| `100` | `1` | `11` | `-` | `10` | `-` | `01` | **Reserved** | Available for custom |
| `100` | `1` | `11` | `-` | `11` | `-` | `01` | **Reserved** | Available for custom |

**Encoding explanation:**
- **[11:10]** selects shift type (00=SRLI, 01=SRAI) or general ALU (10=ANDI, 11=register ops)
- For register ops ([11:10]=11): **[6:5]** selects operation (00=SUB, 01=XOR, 10=OR, 11=AND)
- All operate on **compressed registers** (x8-x15 only) to save encoding space

### Jumps and Branches

| [15:13] | [12:2] | [1:0] | Instruction | Description |
|---------|--------|-------|-------------|-------------|
| `101` | `imm[11\|4\|9:8\|10\|6\|7\|3:1\|5]` | `01` | **C.J** | `pc ← pc + sext(imm)` (unconditional jump ±2KiB) |
| `110` | `imm[8\|4:3], rs1', imm[7:6\|2:1\|5]` | `01` | **C.BEQZ** | `if (rs1' == 0) pc ← pc + sext(imm)` (±256B) |
| `111` | `imm[8\|4:3], rs1', imm[7:6\|2:1\|5]` | `01` | **C.BNEZ** | `if (rs1' ≠ 0) pc ← pc + sext(imm)` (±256B) |

**Notes:**
- C.J replaces short-range unconditional branches
- Branch range: **±256 bytes**  
- Jump range: **±2 KiB**



## Quadrant 2 (`instr[1:0] = 2'b10`) - Stack Operations and Register Moves

**Primary use:** Stack pointer operations, register transfers, returns, and function calls

### C.SLLI (Shift Left Logical Immediate)

| [15:13] | [12] | [11:7] | [6:2] | [1:0] | Instruction | Description |
|---------|------|--------|-------|-------|-------------|-------------|
| `000` | `nzuimm[5]` | `rd≠0` | `nzuimm[4:0]` | `10` | **C.SLLI** | `rd ← rd << uimm` (HINT if rd=0, RV32 NSE if nzuimm[5]=1) |

**Note:** Only full registers (x0-x15) allowed, not just compressed registers

### Stack Loads (SP-relative addressing)

| [15:13] | [12:5] | [4:2] | [1:0] | Instruction | Description |
|---------|--------|-------|-------|-------------|-------------|
| `001` | `uimm[5], rd, uimm[4:3\|8:6]` | `10` | ~~C.FLDSP~~ | NOT IMPLEMENTED (no FP) |
| `010` | `uimm[5], rd≠0, uimm[4:2\|7:6]` | `10` | **C.LWSP** | `rd ← mem32[sp + uimm×4]` (RES if rd=0) |
| `011` | `uimm[5], rd, uimm[4:2\|7:6]` | `10` | ~~C.FLWSP~~ | NOT IMPLEMENTED (no FP) |

**Notes:**
- Load from stack pointer + offset (up to **252 bytes** for C.LWSP)
- Faster than computing address and loading separately
- Reserved encodings prevent accidentally loading to x0 (no effect)

### Register Operations

| [15:13] | [12] | [11:7] | [6:2] | [1:0] | Instruction | Description |
|---------|------|--------|-------|-------|-------------|-------------|
| `100` | `0` | `rs1≠0` | `0` | `10` | **C.JR** | `pc ← rs1` (jump to register, RES if rs1=0) |
| `100` | `0` | `rd≠0` | `rs2≠0` | `10` | **C.MV** | `rd ← rs2` (register copy, HINT if rd=0) |
| `100` | `1` | `0` | `0` | `10` | **C.EBREAK** | Breakpoint exception (trap to debugger) |
| `100` | `1` | `rs1≠0` | `0` | `10` | **C.JALR** | `x1 ← pc+2; pc ← rs1` (call via register) |
| `100` | `1` | `rd≠0` | `rs2≠0` | `10` | **C.ADD** | `rd ← rd + rs2` (HINT if rd=0) |

**Encoding explanation:**
- **Bit [12]** distinguishes between two groups:
  - `0`: Jumps and moves (no writeback to x1)
  - `1`: Calls and adds (may write to x1 or rd)
- **C.JR** with rs1=x1 returns from function (common pattern)
- **C.JALR** saves return address to x1 (function call through pointer)
- **C.MV** is faster than `ADDI rd, rs2, 0` and shorter than `ADD rd, rs2, x0`

### Stack Stores (SP-relative addressing)

| [15:13] | [12:2] | [1:0] | Instruction | Description |
|---------|--------|-------|-------------|-------------|
| `101` | `uimm[5:3\|8:6], rs2` | `10` | ~~C.FSDSP~~ | NOT IMPLEMENTED (no FP) |
| `110` | `uimm[5:2\|7:6], rs2` | `10` | **C.SWSP** | `mem32[sp + uimm×4] ← rs2` (store to stack) |
| `111` | `uimm[5:2\|7:6], rs2` | `10` | ~~C.FSWSP~~ | NOT IMPLEMENTED (no FP) |

**Notes:**
- Store to stack pointer + offset (up to **252 bytes**)
- Used for saving registers to stack during function calls
- Floating-point slots (`001`, `011`, `101`, `111`) = **4 encodings available for custom instructions**




## Quadrant 3 (Standard 32-bit Instructions)

**Not actually a quadrant** - when `instr[1:0] = 2'b11`, the CPU decodes a full **32-bit instruction** instead of a 16-bit compressed one.

### 32-bit Instruction Format Overview

Standard RISC-V instructions use various formats depending on the operation type:

**R-type** (Register-Register):
```
31    25 24  20 19  15 14   12 11   7 6     0
[funct7] [rs2] [rs1] [funct3] [rd] [opcode]
```
Used for: ADD, SUB, AND, OR, XOR, SLT, shifts, MUL

**I-type** (Immediate):
```
31         20 19  15 14   12 11   7 6     0
[imm[11:0]] [rs1] [funct3] [rd] [opcode]
```
Used for: ADDI, ANDI, ORI, XORI, SLTI, loads (LW, LH, LB), JALR

**S-type** (Store):
```
31    25 24  20 19  15 14   12 11   7 6     0
[imm[11:5]] [rs2] [rs1] [funct3] [imm[4:0]] [opcode]
```
Used for: SW, SH, SB (storing to memory)

**B-type** (Branch):
```
31 30    25 24  20 19  15 14   12 11    8 7 6     0
[imm[12|10:5]] [rs2] [rs1] [funct3] [imm[4:1|11]] [opcode]
```
Used for: BEQ, BNE, BLT, BGE, BLTU, BGEU

**U-type** (Upper Immediate):
```
31         12 11   7 6     0
[imm[31:12]] [rd] [opcode]
```
Used for: LUI, AUIPC (loading large constants)

**J-type** (Jump):
```
31 30       21 20 19       12 11   7 6     0
[imm[20|10:1|11|19:12]] [rd] [opcode]
```
Used for: JAL (unconditional jump with return address)

### Why Use 32-bit Instructions?

- **Wider immediates** (up to 20 bits for LUI vs 6 bits for compressed)
- **Full register access** (all x0-x31 in standard RISC-V, x0-x15 in RV32E)
- **Operations not available in compressed set** (e.g., LUI, AUIPC, compare ops)
- **Custom instructions** often use 32-bit encodings for more opcode space

See the "32-bit Instruction Reference" section below for TinyMOA's implemented instructions.

---
## 32-bit Instruction Reference (RV32E Base)

TinyMOA implements the **RV32E** base instruction set, which provides essential arithmetic, logical, memory, and control flow operations using 32-bit encodings.

### Arithmetic and Logical (R-type and I-type)

**Register-Register Operations** (opcode = `01100`)

| Instruction | Opcode[6:2] | funct3 | funct7[5] | Operation | Description |
|-------------|-------------|--------|-----------|-----------|-------------|
| **ADD** | `01100` | `000` | `0` | `rd ← rs1 + rs2` | Add registers |
| **SUB** | `01100` | `000` | `1` | `rd ← rs1 - rs2` | Subtract registers |
| **SLT** | `01100` | `010` | `0` | `rd ← (rs1 < rs2) ? 1 : 0` | Set if less than (signed) |
| **SLTU** | `01100` | `011` | `0` | `rd ← (rs1 < rs2) ? 1 : 0` | Set if less than (unsigned) |
| **AND** | `01100` | `111` | `0` | `rd ← rs1 & rs2` | Bitwise AND |
| **OR** | `01100` | `110` | `0` | `rd ← rs1 \| rs2` | Bitwise OR |
| **XOR** | `01100` | `100` | `0` | `rd ← rs1 ^ rs2` | Bitwise XOR |
| **SLL** | `01100` | `001` | `0` | `rd ← rs1 << rs2[4:0]` | Shift left logical |
| **SRL** | `01100` | `101` | `0` | `rd ← rs1 >> rs2[4:0]` | Shift right logical |
| **SRA** | `01100` | `101` | `1` | `rd ← rs1 >>> rs2[4:0]` | Shift right arithmetic |

**Immediate Operations** (opcode = `00100`)

| Instruction | Opcode[6:2] | funct3 | Operation | Description |
|-------------|-------------|--------|-----------|-------------|
| **ADDI** | `00100` | `000` | `rd ← rs1 + sext(imm)` | Add immediate (-2048 to +2047) |
| **SLTI** | `00100` | `010` | `rd ← (rs1 < sext(imm)) ? 1 : 0` | Set if less than immediate (signed) |
| **SLTIU** | `00100` | `011` | `rd ← (rs1 < imm) ? 1 : 0` | Set if less than immediate (unsigned) |
| **ANDI** | `00100` | `111` | `rd ← rs1 & sext(imm)` | Bitwise AND with immediate |
| **ORI** | `00100` | `110` | `rd ← rs1 \| sext(imm)` | Bitwise OR with immediate |
| **XORI** | `00100` | `100` | `rd ← rs1 ^ sext(imm)` | Bitwise XOR with immediate |
| **SLLI** | `00100` | `001` | `rd ← rs1 << imm[4:0]` | Shift left logical immediate |
| **SRLI** | `00100` | `101` | `rd ← rs1 >> imm[4:0]` | Shift right logical immediate (imm[11:5]=`0000000`) |
| **SRAI** | `00100` | `101` | `rd ← rs1 >>> imm[4:0]` | Shift right arithmetic immediate (imm[11:5]=`0100000`) |

**Notes:**
- All immediates are **sign-extended** to 32 bits (except for unsigned compare)
- Shifts use only **lower 5 bits** of shift amount (0-31 bit positions)
- **SLLI/SRLI/SRAI** distinguished by immediate encoding, not a separate opcode

### Memory Operations

**Load Instructions** (opcode = `00000`)

| Instruction | Opcode[6:2] | funct3 | Operation | Description |
|-------------|-------------|--------|-----------|-------------|
| **LW** | `00000` | `010` | `rd ← mem32[rs1 + sext(imm)]` | Load word (32-bit, sign-extend) |
| **LH** | `00000` | `001` | `rd ← sext(mem16[rs1 + sext(imm)])` | Load halfword (16-bit, sign-extend) |
| **LHU** | `00000` | `101` | `rd ← zext(mem16[rs1 + sext(imm)])` | Load halfword unsigned (zero-extend) |
| **LB** | `00000` | `000` | `rd ← sext(mem8[rs1 + sext(imm)])` | Load byte (8-bit, sign-extend) |
| **LBU** | `00000` | `100` | `rd ← zext(mem8[rs1 + sext(imm)])` | Load byte unsigned (zero-extend) |

**Store Instructions** (opcode = `01000`)

| Instruction | Opcode[6:2] | funct3 | Operation | Description |
|-------------|-------------|--------|-----------|-------------|
| **SW** | `01000` | `010` | `mem32[rs1 + sext(imm)] ← rs2` | Store word (32-bit) |
| **SH** | `01000` | `001` | `mem16[rs1 + sext(imm)] ← rs2[15:0]` | Store halfword (16-bit) |
| **SB** | `01000` | `000` | `mem8[rs1 + sext(imm)] ← rs2[7:0]` | Store byte (8-bit) |

**Notes:**
- Load/store offset range: **±2048 bytes** from base address (12-bit signed immediate)
- **Alignment**: LW/SW require 4-byte alignment, LH/SH require 2-byte alignment
- **Sign-extension**: LH/LB extend sign bit to fill upper bits; LHU/LBU zero-fill

### Upper Immediate Operations

| Instruction | Opcode[6:2] | Operation | Description |
|-------------|-------------|-----------|-------------|
| **LUI** | `01101` | `rd ← imm << 12` | Load upper immediate (sets bits [31:12]) |
| **AUIPC** | `00101` | `rd ← pc + (imm << 12)` | Add upper immediate to PC |

**Usage example:**
```assembly
LUI  x10, 0x12345      # x10 = 0x12345000
ADDI x10, x10, 0x678   # x10 = 0x12345678  (load 32-bit constant)

AUIPC x11, 0x1000      # x11 = pc + 0x1000000  (PC-relative addressing)
```

### Control Flow

**Unconditional Jumps**

| Instruction | Opcode[6:2] | Operation | Description |
|-------------|-------------|-----------|-------------|
| **JAL** | `11011` | `rd ← pc+4; pc ← pc + sext(imm<<1)` | Jump and link (±1MiB range) |
| **JALR** | `11001` | `rd ← pc+4; pc ← (rs1 + sext(imm)) & ~1` | Jump and link register |

**Conditional Branches** (opcode = `11000`)

| Instruction | Opcode[6:2] | funct3 | Operation | Description |
|-------------|-------------|--------|-----------|-------------|
| **BEQ** | `11000` | `000` | `if (rs1 == rs2) pc ← pc + sext(imm<<1)` | Branch if equal |
| **BNE** | `11000` | `001` | `if (rs1 ≠ rs2) pc ← pc + sext(imm<<1)` | Branch if not equal |
| **BLT** | `11000` | `100` | `if (rs1 < rs2) pc ← pc + sext(imm<<1)` | Branch if less than (signed) |
| **BGE** | `11000` | `101` | `if (rs1 ≥ rs2) pc ← pc + sext(imm<<1)` | Branch if greater or equal (signed) |
| **BLTU** | `11000` | `110` | `if (rs1 < rs2) pc ← pc + sext(imm<<1)` | Branch if less than (unsigned) |
| **BGEU** | `11000` | `111` | `if (rs1 ≥ rs2) pc ← pc + sext(imm<<1)` | Branch if greater or equal (unsigned) |

**Notes:**
- **JAL** range: ±1 MiB (20-bit immediate × 2)
- **Branches** range: ±4 KiB (12-bit immediate × 2)
- **JALR** computes target and clears LSB (ensures even address)
- Standard calling convention uses **JAL/JALR with rd=x1** for function calls

### System Instructions

| Instruction | Encoding | Operation | Description |
|-------------|----------|-----------|-------------|
| **EBREAK** | `0x00100073` | Trap to debugger | Breakpoint exception |
| **ECALL** | `0x00000073` | Trap to OS | Environment call (syscall) |
| **FENCE** | `0x0000000F` (opcode=`0001111`) | Memory barrier | Order memory operations |

**Notes:**
- **EBREAK**: Used by debuggers to set breakpoints
- **ECALL**: Transfers control to operating system or monitor
- **FENCE**: Ensures memory operations complete before proceeding (important for I/O, typically encoded as `fence iorw, iorw`)

---