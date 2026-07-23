# GCPU / 68HC11 / Assembly — Condensed Study Guide

## 1. The Big Picture: Computer Functional Block Diagram

Every computer boils down to three talking parts, connected by three kinds of "wires":

- **CPU** (contains ALU + Control Unit)
- **Memory**
- **I/O & Peripherals**

They exchange three signal types:
| Signal | Carries |
|---|---|
| **Data** | The actual values being moved/computed |
| **Address** | *Where* in memory/IO the data comes from or goes |
| **Control** | Read/write, enable, timing signals |

**Memory Unit** interface: Data Out, Data Address, Read/Write, Enable → returns Data In, Status
**I/O Unit** interface: Data Out, Device Address, Control → returns Data In, Device Identity, Status

> Think of Memory and I/O as "vending machines" — CPU sends an address (which slot) and a control signal (buy/read), and gets data back.

---

## 2. Inside the CPU

### Control Unit — the "manager"
**Job:** decode instructions, generate control signals, generate timing.

**Hardware pieces:**
- **Instruction Register (IR)** — holds the current instruction
- **Program Counter (PC)** — address of the *next* instruction
- **Instruction Decoder** — figures out what the instruction means
- **Control Sequence Generator** — issues the actual control signals
- **Clock** — the heartbeat driving everything

**Instruction Register detail:** It's just a MUX + D-flip-flop.
- MUX select line = `IR_LD`
- MUX input 0 = current IR value (hold)
- MUX input 1 = new value from the input bus (load)
- MUX output → D input of flip-flop → this **is** the IR

### ALU — the "worker"
**Job:** arithmetic, logic, shifting, and flag-setting on data.

**Hardware pieces:**
- Arithmetic/Logical circuit
- **Accumulator** (holds operand / result)
- **Shifter**
- **Status Register** (flags)

Data flows: Memory ↔ Accumulator ↔ ALU ↔ Shifter → Status Register flags back to Control Unit.
Function Code (from Control Unit) tells the ALU *which* operation to do.

---

## 3. Programming Models (the registers you can actually use)

### GCPU (simplified teaching CPU)
| Register | Size | Purpose |
|---|---|---|
| A, B | 8-bit | General-purpose accumulators |
| IX, IY | 16-bit | Index registers |
| MAR | 16-bit | Memory Address Register (**hidden**, used internally for extended addressing) |
| PC | 16-bit | Program Counter |
| CCR | 2-bit | Condition Code Register: **N** (negative), **Z** (zero) |
| IR | — | Instruction Register (**hidden**) |

### 68HC11 (real chip — superset of GCPU concepts)
| Register | Size | Purpose |
|---|---|---|
| A, B (or combined as **D**) | 8-bit each / 16-bit combined | Accumulators |
| IX, IY | 16-bit | Index registers |
| SP | 16-bit | Stack Pointer |
| PC | 16-bit | Program Counter |
| CCR | 8-bit | Flags: **S X H I N Z V C** |

**CCR flags (68HC11), left to right:**
S(stop disable) X(interrupt mask) H(half-carry, bit3) I(interrupt mask) N(negative) Z(zero) V(overflow) C(carry/borrow)

**Memory map of our specific 68HC11 (E9 variant):**
- RAM: `$0000–$00FF` and `$0100–$01FF` (512 bytes)
- EEPROM: `$B600–$B7FF` (512 bytes)
- ROM (Buffalo Monitor): `$D000–$FFFF` (12K)
- Internal registers: `$1000–$103F`

**Stack grows DOWN.** On interrupt, PC, Y, X, ACCA, ACCB, CCR get pushed (in that order from SP upward).

> **Key idea — "nearly symmetric" instruction set:** Most 68HC11 instructions work identically on A or B — e.g. `LDA` → `LDAA`/`LDAB`, `STA` → `STAA`/`STAB`, `ROL` → `ROLA`/`ROLB`. Learn the pattern once, apply it to both registers.

---

## 4. Assembly Syntax — the anatomy of one line

```
[Label:]   Operation-Mnemonic   Operand(s)   ; Comment
```

- **Label** — symbolic name for an address (used by branches). Separated from the rest by whitespace or a colon (colon is not part of the name).
- **Operation mnemonic** — the actual command (e.g., `LDAA`)
- **Operand(s)** — 0, 1, 2, or 3 depending on instruction
- **Comment** — after `;`, or the *whole line* is a comment if it starts with `*` in the label field.

### Assemble pipeline
```
Text Editor → Source (.txt/.asm) → Assembler → Object file (machine code) + List file (.lst)
                                                        ↓
                                         Runs on real μP or μP Simulator
```
- **Source code** = assembly language (human-readable)
- **Object code** = machine language (binary/hex — what the μP executes)
- **Assembler** = program that translates source → object

---

## 5. Data Transfer Instructions

Three flavors, based on where data moves:
1. **Register ↔ Register** (internal) — e.g. `TAB`, `TBA`
2. **Memory ↔ Register** — e.g. `LDAA addr`, `LDAB #data`
3. **I/O device ↔ Register** — e.g. `LDAA memory-mapped-io`

### Inherent Addressing (register-to-register)
No memory reference needed — everything is "inherently" known by the CPU.

- `Twz` pattern: Transfer register **w** → register **z** (e.g. `TAB` = Transfer A to B)
- `TAP` : CCR ← A
- `TPA` : A ← CCR
- `XGDX` / `XGDY` : swap D ↔ X (or Y) — 68HC11 only, not on GCPU

| Instruction | Effect | Flags affected |
|---|---|---|
| TAB/TBA | B←A or A←B | N,Z,V |
| TAP | CCR←A | All |
| TPA | A←CCR | None |
| XGDX/XGDY | D↔X or D↔Y | None |

---

## 6. Addressing Modes — **the most important section**

**Effective Address (EA)** = where the data actually comes from/goes to. Every addressing mode is just a different *recipe* for computing the EA.

### (a) Direct Addressing — "page 0 shortcut"
- Only supply the **low byte**; high byte is assumed `$00`.
- Only works for addresses `$0000–$00FF`.
- **Why use it:** faster (1 less memory read) and saves 1 byte of program space.
- Example: `LDAA $37` → EA = `$0037`

### (b) Extended Addressing — "full address"
- Supply the **complete 16-bit address**.
- Works anywhere in memory (`$0000–$FFFF`).
- Example (68HC11): `LDAA $4237` → machine code `b6 42 37`
- GCPU has **no** direct addressing mode — it always uses extended (or needs immediate/indexed).

**⚠️ Byte order matters (Endianness):**
| | GCPU | 68HC11 |
|---|---|---|
| Type | Little-endian | Big-endian |
| `LDAA $4237` machine code | `04 37 42` (low byte first) | `b6 42 37` (high byte first) |

### (c) Indexed Addressing — "base + offset"
```
Effective Address = (value in IX or IY)  +  (8-bit UNSIGNED offset)
```
- IX/IY itself is **never modified** — it's just used as a base.
- Offset range: `0` (`$00`) to `255` (`$FF`) — **cannot be negative**.
- Great for arrays/tables of contiguous memory (e.g., looping through digits of a phone number).
- Example: If `IX = $1234`, then `LDAA $2A,X` → EA = `$1234 + $2A = $125E`
- **Big advantage:** to process a *different* block of data, just reload the index register — the same instruction sequence still works.

### (d) Immediate Addressing — "the value IS the operand"
- Data is embedded directly in the instruction, marked with `#`.
- EA = address of that embedded number itself (in the code stream), not some other memory cell.
- Number bases:
  | Prefix | Base |
  |---|---|
  | (none) | decimal |
  | `$` | hex |
  | `@` | octal |
  | `%` | binary |
  | `'` | single ASCII char |
- Example: `LDAA #$34` → loads the literal value `$34` into A.

### (e) Inherent Addressing (recap)
- No operand/address needed at all — instruction is self-contained (e.g. `TAB`, `SUM_BA`/`ABA`).

### Quick comparison table
| Mode | Operand gives you... | Range | Notes |
|---|---|---|---|
| Direct | low byte only | `$0000–$00FF` | fast, compact, "page 0" |
| Extended | full 16-bit address | `$0000–$FFFF` | GCPU's only memory-addressing mode besides indexed/immediate |
| Indexed | offset added to IX/IY | anywhere, offset 0–255 | for arrays; index register unchanged |
| Immediate | the data itself | n/a | prefixed with `#` |
| Inherent | nothing | n/a | register-to-register only |

---

## 7. Assembler Directives (pseudo-instructions)

These are commands **to the assembler**, not to the CPU — they don't generate CPU instructions (except indirectly via space/values).

| Directive | Purpose | Example |
|---|---|---|
| `ORG` | Set the location counter (where following code/data starts) | `ORG $7300` |
| `EQU` | Give a symbolic name a permanent value (text substitution) | `PI EQU 3` |
| `DC.B` | Define Constant Byte(s) — allocates **and** initializes, 1 byte each | `Table DC.B 3,9,44,$2E` |
| `DC.W` | Define Constant Word(s) — same but 2 bytes each | `Signal DC.W $5000,$1000` |
| `DS.B` | Define Storage Byte(s) — allocates space, **no** initialization (for variables) | `Var1 DS.B 1` |
| `DS.W` | Define Storage Word(s) — same, 2-byte chunks | `Var2 DS.W 2` |

**Key distinction:** `DC.*` = constants (pre-loaded values). `DS.*` = scratch variables (empty space reserved).

**EQU gotcha:** normally a label's value = its *address*. But when a label appears in the label field of an `EQU`, its value = the operand's value instead (not an address).

---

## 8. Self-Check Questions (use these to test yourself)

1. Can you compute the **effective address** for each of the 5 addressing modes, cold, without notes?
2. Why can't Direct Addressing be used above `$00FF`?
3. Why is the GCPU little-endian relevant when reading raw machine code bytes?
4. If `IX = $2000` and you execute `LDAA $05,X`, what's the EA? (Answer: `$2005`)
5. What's the difference between `DC.B 5` and `DS.B 5`?
6. How would you build a "decrement" (`DECA`-equivalent) on the GCPU, which has no native decrement instruction? *(Hint from slides: use 2's complement addition — negate the "1" you want to subtract, then add it.)*

---

## 9. One-Sentence Takeaways

- **CPU = Control Unit (decides) + ALU (computes)**, talking to Memory/IO over Data/Address/Control lines.
- **Addressing mode = the recipe for finding the effective address** — this is the single most testable concept in this file.
- **GCPU is little-endian; 68HC11 is big-endian** — byte order flips depending on which chip you're assembling for.
- **68HC11's instruction set is "nearly symmetric"** — learn one mnemonic pattern, it usually applies to both A and B.
- **`DC` = constants with values; `DS` = empty storage for variables.**