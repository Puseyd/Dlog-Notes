# Moore & Mealy Machines, ASM Design, ASM with ROMs
### EEL3701 — Files 17, 18, 20 (Drs. Schwartz & Arroyo)

---

# PART 1: Moore & Mealy Machines (File 17)

## Classical State Machine Design — 4 Steps
1. **Draw the State Diagram** (Moore or Mealy)
2. **Assign State Identifiers** (binary patterns/names)
3. **Pick FFs**, use excitation tables → get FF-input equations
4. **Obtain FF-input & output equations** (K-Maps → MSOP)

## Moore vs. Mealy
| | Moore Machine | Mealy Machine |
|---|---|---|
| Output depends on | **state only**: Z = g(Qi) | **state + input**: Z = g(Qi, X) |
| Next state | Qⱼ⁺ = f(Qi, X) | Qⱼ⁺ = f(Qi, X) |
| States needed | Often more | **Often fewer** (can merge states that Moore must separate) |
| Output timing | Output lags input by up to 1 clock (changes only *after* arriving in new state) | Output can change **asynchronously**, between clock pulses (spurious pulses possible) |

## Example: Sequence Detector for X = 0 1 0* 1
(where 0* = zero or more 0's; e.g. accepts 011, 0101, 01001... but not 0111)

**Design logic (state-by-state reasoning):**
- S0: "looking for first 0" (start)
- S1: "got the 0, looking for 1"
- S2: "got 01, looking for another 1 to succeed with 011" — or 0 to go looking for 0*1
- S4: "success — just detected {011}" (Moore output Z=1)
- S3: "in the middle of 0 1 0* ..., looking for closing 1"
- S5: "success — just detected {010...01}" (Moore output Z=1)
- From S4/S5, restart appropriately depending on next input (1→S0, 0→S1)

**State assignment:** 6 states → need **3 flip-flops** (2²=4 < 6 ≤ 2³=8); 2 unused codes become **don't cares** in K-maps.
- *Heuristic:* try to change only 1 bit between adjacent states (Gray-code-like), though not required in EEL3701.

**Mealy reduces state count** — same problem needs only **5 states** in Mealy vs. 6 in Moore, because Mealy can fold the "success" indication into a transition instead of a separate state.

## Implementation (D Flip-Flops)
- Using K-maps on the (X, Q2, Q1, Q0) truth table → derive:
  - Q2⁺ = Q2Q0 + Q2/X + Q1Q0/X
  - Q1⁺ = Q2 + Q1Q0 + Q0X
  - Q0⁺ = /Q1Q0 + /X
  - **Z_Moore = Q1/Q0** (function of state only)
  - **Z_Mealy = Q2X + Q1Q0X** (function of state AND input)

**Key hardware difference:**
- **Moore:** Z comes straight off the state FFs → Z changes synchronously, lags input by 1 clock.
- **Mealy:** Z comes off a combinational gate driven by both FFs and X → Z can glitch ("spurious pulses") whenever X changes between clocks. Harmless if the receiving circuit is itself clocked.

---

# PART 2: ASM (Algorithmic State Machine) Design (File 18)

## ASM Notation — Only 3 Symbols!
| Symbol | Meaning |
|---|---|
| **Rectangle (State box)** | A state; name in one corner, state bits in the other; unconditional ("Moore") outputs listed inside |
| **Diamond (Decision)** | A conditional branch on an input or input function (e.g., X, or X=A·B) |
| **Oval (Conditional output)** | A "Mealy"-style output, placed in a path — asserted only when that path is taken |

- **Rule:** In a *final* ASM, only **true** outputs go inside state boxes (don't write "C=0" if it's always false in that state — leave it out; absent = false by default).
- Never start circuit design without first drawing the ASM chart.
- Free tool recommendation: **app.diagrams.net** (draw.io) for ASM charts.

## Conditional (Mealy) Outputs — Spurious Pulses
- If X changes between clock edges, a conditional output changes too ("spurious pulse").
- Fine as long as whatever reads that output is **also clocked** — by the time the next clock edge arrives, everything (X, Z, f(Z)) is stable again.

## Why Use an ASM (vs. plain state diagram)?
- Same 6 realization choices as any combinational/sequential design: **(1) gates/K-maps, (2) PLD, (3) MUX, (4) ROM, (5) other LSI, (6) μP/μC.**
- ASM charts scale much better than state diagrams when there are many inputs (a state diagram needs 2ⁿ arrows out of each state for n inputs — an ASM chart avoids this explosion).

## Worked Example: Washing Machine Controller
**Inputs:** STRT, SHOT, FULL, EMP, TO
**Outputs:** OHOT, PUMP, FILL, AG, STIME, SPIN, STRTOFF

**Sequence:** Idle → Fill (hot/cold per SHOT) → Agitate (until timer) → Drain → Rinse 1 → Rinse 2 → Agitate → Drain → Spin → back to Idle. STRT held "off" at any point ⇒ machine **holds state** until STRT is on again.

**ROM Realization (Step 3):**
- States: Idle(000), Agitate(001), Drain(010), Rinse(011), Rinse2(100), SpinIt(101)
- Needed: **256×10 ROM** (8 address bits: 5 inputs + 3 state bits; 10 data bits: 7 outputs + 3 next-state bits)
- **Reduction trick:** if some outputs are simple functions of *other* signals already known (e.g., STRTOFF and SPIN both only depend on being in state SpinIt + STRT/TO), compute them with small gates **outside** the ROM instead of storing them → shrinks ROM to **256×8**.

## Two Key Skills

### Skill 1: ASM Chart → Timing Diagram
- Walk state by state; at each state, note inputs that matter and outputs asserted; draw as waveforms across clock periods (states change once per clock).

### Skill 2: Timing Diagram → ASM Chart (reverse-engineering)
- Read outputs/inputs at each time step, infer state transitions.
- **Key trick:** when a transition isn't explicitly exercised in the diagram, choose the **simplest/most reduced conditional** consistent with the data (e.g., collapse a 2-input decision down to 1 input if only one bit ever actually matters, using don't-cares).
- Example resulted in: unconditional B→A transition; C's Count.En output turned out to be **conditional** (only true when Buf.Full=1) even though Reg.LD stayed unconditional.

---

# PART 3: ASM with ROMs (File 20)

## General Method: ASM Chart → Next-State Truth Table → ROM

**Example 2 controller** (states A,B,C,D; inputs In.Bit, Buf.Full; state bits Q1Q0):

### Step A — Build raw next-state table
List every input combination per state → get Q1⁺Q0⁺ and outputs (Count.En, Reg.LD, Out.Flag).

### Step B — Two ways to organize the table
1. **With wildcards/rearranging (compact):** 6 rows using "*" for don't-care inputs — easier to read but not directly ROM-loadable.
2. **Full expansion (no wildcards):** All 2⁴=16 rows (2 inputs × 2 state bits) — this is what actually gets burned into the ROM.

### Step C — Determine ROM size
- **# address lines = (# inputs) + (# state bits)** → here 2+2 = 4 → 2⁴=16 addresses.
- **# data lines = (# state bits, i.e. FF excitation) + (# outputs)** → here 2+3 = 5.
- ⇒ Need a **16×5 ROM**.

### Step D — Functional Block Diagram
- ROM address inputs: inputs (In.Bit, Buf.Full) + fed-back state bits (Q1, Q0).
- ROM data outputs: next-state bits (D1=Q1⁺, D0=Q0⁺) feed back into D-FFs; other data outputs (Out.Flag, Count.EN, Reg.LD) go straight out.
- **Practical tip:** the raw truth table order from design work often isn't ROM-friendly — reorder/rearrange it purely for convenience when actually programming the ROM.

## Alternative: MUX Implementation
- Instead of a ROM, use **multiplexers** to realize the same next-state/output equations.
- Typical setup: one MUX per state bit (D1, D0), each with **≥4 inputs** selected by state bits (Q1,Q0), and data inputs being functions of the remaining inputs (In.Bit, Buf.Full).
- If you use **3 select lines** (e.g., In.Bit, Q1, Q0) you get an 8-input MUX where each data input reduces to a function of just the remaining variable (e.g., Buf.Full).
- Same approach extends to output generation: e.g., 3 more MUXes for Out.Flag, Reg.LD, Count.En — note some outputs (like Count.En) may depend on an extra variable (Buf.Full) while others don't.

## Second Example (4 inputs: W, X, Y, Z; states P, Q, R, S)
- Next-state table has many "don't cares" because most state transitions are only reachable one specific way (e.g., only R→P is possible; R→Q, R→R, R→S can't occur).
- **Size math:** 4 inputs + 2 state bits = 6 address bits → **64×2 ROM** for just the 2 state-bit outputs, or **64×4 ROM** if also generating 2 extra outputs directly.
- MUX equations derived directly by inspecting which table rows make an output = 1, e.g.:
  - MUX0(0) = /X·Y + X = X + Y
  - MUX1(0) = /X·/Y + X = X + /Y
  - (remaining MUX equations read directly off the table in the same way)

## General Pattern (applies to any ASM→hardware conversion)
| Design choice | Address bits needed | Data bits needed |
|---|---|---|
| ROM | inputs + state bits | state bits (next-state) + outputs |
| MUX | select lines = state bits (or fewer, if some inputs used as MUX select) | one MUX per output/next-state bit; data inputs = remaining input functions |

---
*Source: EEL3701 Lecture Files 17, 18, 20 — University of Florida, Drs. Schwartz & Arroyo*