
## Custom Extensions (TinyMOA-specific)

TinyMOA extends the base RISC-V instruction set with custom instructions for efficient CIM operations and firmware optimization.

### Zicond Extension (Conditional Zero)

| Instruction | Encoding | funct7 | funct3 | Operation | Description |
|-------------|----------|--------|--------|-----------|-------------|
| **CZERO.EQZ** | R-type | `0000111` | `101` | `rd ← (rs2 == 0) ? 0 : rs1` | Zero if rs2 is zero |
| **CZERO.NEZ** | R-type | `0000111` | `111` | `rd ← (rs2 ≠ 0) ? 0 : rs1` | Zero if rs2 is non-zero |

**Usage:** Conditional moves without branches
```assembly
CZERO.EQZ x10, x11, x12  # x10 = (x12 == 0) ? 0 : x11
```

### Thread/Global Pointer Access (Compressed)

TinyMOA hardcodes **gp (x3) = 0x1000400** and **tp (x4) = 0x8000000** for fast memory access.

| Instruction | Format | Operation | Description |
|-------------|--------|-----------|-------------|
| **C.LWTP** | `[15:10]=100111, [4:2]=rd', [1:0]=10` | `rd' ← mem32[tp + uimm×4]` | Load from thread pointer (0-252B offset) |
| **C.SWTP** | `[15:10]=100111, [4:2]=rs2', [1:0]=10` | `mem32[tp + uimm×4] ← rs2'` | Store to thread pointer |

**Notes:**
- **tp window**: 0x8000000 - 0x80000FF (256 bytes for peripheral access)
- **gp window**: 0x1000400 - 0x10007FF (1KB for fast RAM access)

### Context Save/Restore

| Instruction | Format | Operation | Description |
|-------------|--------|-----------|-------------|
| **C.SCXT** | `[15:10]=001000, [9:7]=rs1', [6:2]=rs2', [1:0]=00` | Save x8-x15 to [rs1'] | Save all compressed regs |
| **C.LCXT** | `[15:10]=100011, [9:7]=rs1', [6:2]=rd', [1:0]=10` | Load x8-x15 from [rs1'] | Restore compressed regs |

**Usage:** Fast context switching between threads or interrupt handlers
```assembly
C.SCXT x8, x9    # Save x8-x15 to memory pointed by x8
C.LCXT x8, x9    # Restore x8-x15 from memory pointed by x8
```

### 16-bit Multiply

| Instruction | Format | Operation | Description |
|-------------|--------|-----------|-------------|
| **C.MUL16** | `[15:10]=100100, [9:7]=rs1'/rd', [6:5]=10, [4:2]=rs2', [1:0]=10` | `rd' ← rs1'[15:0] × rs2'[15:0]` | 16×16 → 32-bit multiply |

**Note:** Faster and smaller than full 32×32 multiply for DSP/CIM operations

---

## Register Conventions (RISC-V ABI)

TinyMOA uses **x0-x15** (RV32E embedded). Register usage follows standard RISC-V calling conventions:

| Register | ABI Name | Usage | Saved by |
|----------|----------|-------|----------|
| x0 | zero | Hardwired to 0 (reads always return 0, writes ignored) | - |
| x1 | ra | Return address (link register) | Caller |
| x2 | sp | Stack pointer | Callee |
| x3 | gp | Global pointer (TinyMOA: **0x1000400**) | - |
| x4 | tp | Thread pointer (TinyMOA: **0x8000000**) | - |
| x5 | t0 | Temporary register | Caller |
| x6-x7 | t1-t2 | Temporary registers | Caller |
| x8 | s0/fp | Saved register / Frame pointer | Callee |
| x9 | s1 | Saved register | Callee |
| x10-x11 | a0-a1 | Function arguments / return values | Caller |
| x12-x15 | a2-a5 | Function arguments | Caller |

**Caller-saved:** Function can freely modify these (caller must save if needed)  
**Callee-saved:** Function must preserve these (callee must save/restore)

### Compressed Register Subset (`rd'`, `rs1'`, `rs2'`)

Compressed instructions encode registers in **3 bits** instead of 4/5, limiting them to **x8-x15**:

| 3-bit encoding | Register | ABI Name | Calculation |
|----------------|----------|----------|-------------|
| `000` | x8 | s0/fp | 8 + 0 = x8 |
| `001` | x9 | s1 | 8 + 1 = x9 |
| `010` | x10 | a0 | 8 + 2 = x10 |
| `011` | x11 | a1 | 8 + 3 = x11 |
| `100` | x12 | a2 | 8 + 4 = x12 |
| `101` | x13 | a3 | 8 + 5 = x13 |
| `110` | x14 | a4 | 8 + 6 = x14 |
| `111` | x15 | a5 | 8 + 7 = x15 |

**Why this subset?** These are the **most frequently used registers** in leaf functions (arguments, saved registers, temporaries). The 3-bit value is added to 8 to get the actual register number.

---

## Beginner's Glossary

**Instruction Encoding Terms:**

- **opcode**: Primary operation code (identifies instruction family)
- **funct3**: 3-bit function code (selects specific operation within family)
- **funct7**: 7-bit function code (further distinguishes operations, e.g., ADD vs SUB)
- **rd**: Destination register (where result is written)
- **rs1**: Source register 1 (first operand)
- **rs2**: Source register 2 (second operand)
- **imm**: Immediate value (constant embedded in instruction)

**Immediate Types:**

- **sext(imm)**: Sign-extend immediate (copy sign bit to fill upper bits)
- **zext(imm)**: Zero-extend immediate (fill upper bits with 0)
- **uimm**: Unsigned immediate (always zero-extended)
- **nzimm**: Non-zero immediate (value=0 is reserved/illegal)

**Memory Terms:**

- **mem32[addr]**: Read/write 32-bit value at address (4-byte access)
- **mem16[addr]**: Read/write 16-bit value at address (2-byte access)
- **mem8[addr]**: Read/write 8-bit value at address (1-byte access)
- **Alignment**: Address must be multiple of access size (addr % 4 = 0 for word access)

**Bit Operations:**

- **<<**: Shift left (multiply by power of 2)
- **>>**: Shift right logical (divide by power of 2, unsigned)
- **>>>**: Shift right arithmetic (divide by power of 2, signed - preserves sign bit)
- **&**: Bitwise AND (mask operations)
- **|**: Bitwise OR (set bits)
- **^**: Bitwise XOR (toggle bits, also used for equality check: a^b == 0 means a == b)

**Other Terms:**

- **pc**: Program counter (address of current instruction)
- **ABI**: Application Binary Interface (calling convention for functions)
- **Hint instruction**: No-op in this implementation, may have meaning in future CPUs
- **RES (Reserved)**: Illegal encoding that will cause trap/exception
- **NSE (Non-Standard Encoding)**: Implementation-specific extension to standard

---

## Quick Reference: Common Patterns

### Loading 32-bit Constants

```assembly
LUI  x10, 0x12345      # x10[31:12] = 0x12345
ADDI x10, x10, 0x678   # x10 = 0x12345678
```

### Function Call

```assembly
JAL  ra, function      # Call function, save return address to ra (x1)
...
JALR zero, ra, 0       # Return (jump to ra, don't save new return address)
```
Or with compressed instructions:
```assembly
C.JAL function         # Equivalent to JAL ra, function
...
C.JR ra                # Equivalent to JALR zero, ra, 0
```

### Conditional Execution (without branches)

```assembly
# result = (condition) ? a : b
CZERO.EQZ x10, x11, x12   # x10 = (x12 == 0) ? 0 : x11  (a if condition)
CZERO.NEZ x13, x14, x12   # x13 = (x12 != 0) ? 0 : x14  (b if !condition)
OR        x10, x10, x13   # x10 = a or b (only one is non-zero)
```

### Stack Frame Setup

```assembly
C.ADDI16SP -32         # Allocate 32 bytes on stack
C.SW   ra, 28(sp)      # Save return address
C.SW   s0, 24(sp)      # Save frame pointer
...
C.LW   ra, 28(sp)      # Restore return address
C.LW   s0, 24(sp)      # Restore frame pointer
C.ADDI16SP 32          # Deallocate stack frame
C.JR   ra              # Return
```

---
