# RAM, ROM — Condensed Notes
### EEL3701 — File 16 (Drs. Schwartz & Arroyo)

---

## 1. Static RAM (SRAM) Basics
- **Model:** long vector of registers, each with an **address** (usually in HEX).
- **Example: 1k × 8 RAM** → 1024 words, 8 bits/word (1 byte). 1K = 1024 bytes = 2¹⁰ bytes.
- **Prefixes/postfixes for number bases:** H/B/O/D (postfix) = Hex/Binary/Octal/Decimal; $/%/@ (prefix) = Hex/Binary/Octal.
- **Bit/byte facts:** 8 bits = 1 byte, 4 bits = 1 nibble; k=2¹⁰, M=2²⁰, G=2³⁰, T=2⁴⁰.

## 2. RAM Pinout & Control Signals
- To address 1k (2¹⁰) locations → need **10 address lines** (A9–A0); 8 data lines (D7–D0).
- Address range example: `0000000000B`–`1111111111B` = `000H`–`3FFH`.
- General RAM: **n address lines, m data lines, control lines** (CE/CS = Chip Enable/Select, WE = Write Enable, OE/RD = Output Enable/Read).
- Data lines are typically **bi-directional** (time-multiplexed).

### RAM truth table (with OE)
| Operation | CE | WE | OE | D |
|---|---|---|---|---|
| Disable | 0 | – | – | Hi-Z |
| Disable | 1 | 0 | 0 | Hi-Z |
| Read | 1 | 0 | 1 | Out |
| Write | 1 | 1 | – | In |

- **Volatile**: info retained only while powered.
- Access times: SRAM ~10–100 ns (CMOS); faster/smaller-capacity options exist (TTL).
- Examples: 2114A (1k×4 SRAM, ~100ns, addr 0–$3FF); 32k×8 SRAM (7–100ns, addr 0–$7FFF).

## 3. Simplified SRAM (No OE)
- WE has higher priority than OE → tie OE to GND (always enabled).
- Behavior: **WE=0 → Read; WE=1 → Write** (when CE active).

## 4. SRAM Timing
- **Read cycle:** address stable → CE(L) active → data valid after access time.
- **Write cycle:** address stable → CE(L) active → WE(L) pulses low → data must be stable/held around WE transition.
- Key timing params: t_A (address access), t_RC (read cycle), t_WC (write cycle), t_SU/t_H (setup/hold for CE, WE, data).

## 5. RAM Expansion
- **Widen data (1k×4 → 1k×8):** use two 1k×4 chips in parallel, same address/CS/WE, split data bus (D7-4 and D3-0).
- **Extend depth (1k×4 → 2k×4):** use two 1k×4 chips; extra address bit A10 selects chip via CS (direct to one chip, inverted to the other).
- Combining both techniques → build larger word-size **and** larger depth arrays (e.g., 2k×8 from 1k×4 chips using both techniques together).

## 6. RAM Connected to a Microprocessor
- Example: 4-byte×8 RAM + μP with 3 address pins (A2-A0), 8 data pins, control lines R, W, E.
- **OE = E·R**, **WE = E·W**; connect D7-D0 directly.
- Only 2 of 3 address lines are needed for the RAM's internal addressing; the 3rd (or its complement) becomes **CS**.
- **4 possible choices** of which address line becomes CS, each placing the RAM at a different position in the μP's address space (e.g., Choice 4 uses CS=/A2 → RAM occupies addresses 000–011, the "low" range).

### Key rule (Conclusions)
- For **contiguous addressing**, always wire the **low-order address bits directly** to the RAM's address pins.
- CS = f(unused/high-order address lines). With **m unused lines → 2ᵐ possible starting addresses** for that RAM block.
- Choosing CS = /A(high bit) → RAM sits in low memory range; CS = A(high bit) → high memory range.

## 7. RAM as Combinational Logic / Equation Implementation
- A RAM can implement any truth table: n address lines = n inputs, m data lines = m outputs.
- Example: 3-input, 8-output device — load the RAM with the truth table's output values as "data" at each address.
- e.g., D5 = /A2·(A1+A0), D4 = /A2, D1 = /A2·/A1·A0 (derived directly from the data pattern stored at each address).

## 8. RAM Internal Data Organization (Matrix)
- Internally organized as row × column, e.g., 8×512×8-bit = (8×512)×8 = 4k×8 RAM, using 9 row bits + 3 column bits (512=2⁹, 8=2³, 4k=2¹²).
- Address range: `$000`–`$FFF` (2¹² = 4096 words).
- User doesn't see the internal row/column split — only sees the final 4k×8 capacity.

## 9. Dynamic RAM (DRAM)
- Also **volatile**, but must be **periodically refreshed** or data is lost.
- Higher density than SRAM, "faster" per-cell but refresh overhead can make net access slower.
- Capacities: e.g., 16M×8 = 4k×4k = 2²⁴ and larger.
- Uses **RAS** (Row Address Strobe) and **CAS** (Column Address Strobe) to time-multiplex address bits — n bits for row + n bits for column = 2n total address bits, sent over the **same pins** sequentially.
- Example: 64k×1 DRAM → 16-bit address split into 8-bit row + 8-bit column (64k = 2¹⁶ = 2⁸×2⁸). Row address sent first, then column address.
- Requires a **DRAM controller**: sequential circuit converting parallel address into 2 multiplexed 8-bit halves.

### DRAM Timing Cycles
- **Read cycle:** Row address latched via RAS(L), then column address latched via CAS(L); WE(L) stays high; Dout goes from high-Z to valid data.
- **Write cycle:** Similar row/column sequencing; WE(L) pulses low during/after CAS to write; Din must be stable.
- **RAS-only refresh cycle:** Only row address + RAS(L) pulsed (no CAS, no data out) — used purely to refresh a row without accessing data.

## 10. Read-Only Memory (ROM) — Types
| Type | Description |
|---|---|
| **ROM (masked)** | Content set at manufacture; non-reversible |
| **PROM** | Programmable — user "blows" fuses (like PLA/PAL); permanent once programmed |
| **EPROM** | Programmed via high-voltage pulses; erased via UV light exposure (5–15 min). Ex: 2708 (1k×8), 2764 (8k×8) |
| **EEPROM** | Electrically erasable/programmable via voltage pulses; erase ~1ms/row or per item. Ex: 2864 (8k×8) |
| **FLASH EEPROM** | Newer, fully electrically alterable. Ex: 28F256 (32k×8) |

- **All ROM types are non-volatile.**

## 11. ROM Model & Pinout
- General: n address lines, m data lines, control (CS/CE, OE).
- After programming, **data bus is output-only** (no write in normal read operation).
- Example: 8k×8 ROM → 13 address lines (A12-A0), 8 data lines.

### ROM truth table
| Function | CE | OE | D |
|---|---|---|---|
| Disable | 0 | – | Hi-Z |
| Disable | – | 0 | Hi-Z |
| Read | 1 | 1 | Out |

- Common use: storing non-changing software (e.g., BIOS/bootstrap/diagnostic code in PCs).

## 12. Example Devices

**2716 EPROM (2k×8):** 11 address lines, 8 data, CE/PRG, OE, Vpp (+25V program voltage), Vcc (+5V).

| Operation | CE/PRG | OE | Vpp | D |
|---|---|---|---|---|
| Read | L | L | +5V | Out |
| Standby | H | – | +5V | Hi-Z |
| Program | L→H | H | +25V | In |
| Program Verify | L | L | +25V | Out |
| Program Inhibit | L | H | +25V | Hi-Z |

**2864 EEPROM (8k×8):** 13 address lines, 8 data, CE, OE, WE.

| Operation | CE | OE | WE | D |
|---|---|---|---|---|
| Read | L | L | H | Out |
| Write | L | H | L | In |
| Standby/Write Inhibit | H | – | – | Hi-Z |
| Output Disable | – | H | – | Hi-Z |
| Chip Erase | L | 12V | L | Hi-Z |

## 13. Application Example: BCD-to-7-Segment Converter via ROM
- 4 address inputs (B3-B0, the BCD digit) → 7 data outputs (Z6-Z0, segments a–g).
- Truth table stored in ROM directly maps each BCD input combination to its 7-segment display pattern (a classic use of ROM as combinational logic lookup).

---
*Source: EEL3701 Lecture File 16 — "RAM, ROM" (University of Florida, Drs. Schwartz & Arroyo)