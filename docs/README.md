
## TinyMOA ISA

TinyMOA uses RV32EC. The RV32E base stands for "RISC-V 32b Embedded" and only uses 16 registers instead of 32, while the "C" extension stands for compressed, adding 16b instructions.

Additionally, TinyMOA extends the base RISC-V instruction set with custom instructions for efficient CIM operations and firmware optimization.

### Custom Instructions
WIP


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

## RV32E Register Conventions (RISC-V ABI)

Since TinyMOA uses RV32E with 16 registers (`x0-x15`) instead of 32, register usage follows standard but adjusted RISC-V calling conventions:

| Register | ABI Name | Usage | Saved by |
|----------|----------|-------|----------|
| x0 | zero | Hardwired to 0 (reads always return 0, writes ignored) | - |
| x1 | ra | Return address (link register) | Caller |
| x2 | sp | Stack pointer | Callee |
| x3 | gp | Global pointer | - |
| x4 | tp | Thread pointer | - |
| x5-7 | t0-t2 | Temporary registers | Caller |
| x8 | s0/fp | Saved register / Frame pointer | Callee |
| x9 | s1 | Saved register | Callee |
| x10-x11 | a0-a1 | Function arguments / return values | Caller |
| x12-x15 | a2-a5 | Function arguments | Caller |

**Caller-saved:** Function can freely modify these (caller must save if needed)  
**Callee-saved:** Function must preserve these (callee must save/restore)

### Compressed Register Subset

Compressed instructions encode registers in 3 bits instead of 4 or 5, limiting them to `x8-x15`. They are written with an apostrophe such as `rd'`, `rs1'`, `rs2'`. Below is the compressed register table:

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

These are the most frequently used registers in leaf functions (arguments, saved registers, temporaries) since the 3-bit value is added to 8 to get the actual register number.

## Glossary
WIP
