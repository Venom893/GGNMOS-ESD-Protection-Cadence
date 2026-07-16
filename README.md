# GGNMOS ESD Protection Cell — Cadence Virtuoso (gpdk090)



![Cadence](https://img.shields.io/badge/Cadence-Virtuoso%20IC6.1.7-E2231A?style=flat-square)




![Spectre](https://img.shields.io/badge/Simulator-Spectre-1f6feb?style=flat-square)




![PDK](https://img.shields.io/badge/PDK-gpdk090%20·%2090nm-2ea043?style=flat-square)




![Domain](https://img.shields.io/badge/Domain-ESD%20Protection%20%2F%20Analog%20I·O-8957e5?style=flat-square)



> Every chip needs ESD protection, or it doesn't survive its first handshake with reality.

A **Gate-Grounded NMOS (GGNMOS)** primary ESD clamp — designed, simulated, characterized, and debugged from a blank schematic in Cadence Virtuoso. Along the way: a real PDK bug found and fixed, a PDK modeling limitation exposed by experiment, a behavioral macro-model built to recover the missing bipolar physics, a Verilog-A clamp that captures full negative-resistance fold-back, and a 2 kV HBM transient stress test.

---

## Why this matters

Before a chip is ever powered on, it gets handled, shipped, and assembled — and every one of those steps can dump thousands of volts of static onto a pin in nanoseconds. Without an on-die clamp, that energy walks straight into the gate oxide and the chip is dead on arrival.

The GGNMOS is the **workhorse primary ESD clamp** used on real I/O pads in production silicon. It sits quietly on the pad during normal operation and turns into a high-current shunt to ground the instant a discharge hits. Understanding and building one is foundational analog/I-O design work — which is exactly why this project starts there.

---

## At a glance

| | |
|---|---|
| **Tool** | Cadence Virtuoso IC6.1.7 |
| **Simulator** | Spectre Circuit Simulator |
| **PDK / Node** | gpdk090 — 90 nm |
| **Analyses** | Transient (ESD-style pulse, HBM) · DC (current-forced snapback sweep) |
| **Device** | Gate-Grounded NMOS primary clamp + behavioral macro-model + Verilog-A snapback clamp |
| **Status** | Transient verified · snapback fully characterized · Verilog-A clamp captures negative-resistance fold-back (Vt1 = 9.0 V, Vh ≈ 5.35 V) · 2 kV HBM transient verified |

---

## Design

| File | Description |
|------|-------------|
| **[GGNMOS cell](Schematics/GGNMOS_ESD_schematic.png)** | NMOS with gate, source, and bulk tied to ground — the configuration that arms the parasitic conduction path during a discharge while keeping the device off in normal use. |
| **[Transient testbench](Schematics/GGNMOS_ESD_tb_schematic.png)** | Pulse source driving the device through the PAD node, emulating a fast-rising ESD-like stress event. |
| **[Snapback testbench](Schematics/GGNMOS_ESD_snapback_tb_schematic.png)** | DC current source forcing current into the PAD node — the correct stimulus for tracing a snapback I–V curve. |
| **[Macro-model cell](Schematics/GGNMOS_ESD_macro_schematic.png)** | GGNMOS with an **explicit parasitic NPN + substrate resistance (Rsub) + avalanche current source** — a behavioral model of the bipolar mechanism the compact model omits. |
| **[Macro-model testbench](Schematics/GGNMOS_ESD_macro_tb_schematic.png)** | Current-forced testbench driving the macro-model to extract its trigger characteristic. |
| **[Behavioral clamp schematic](Schematics/schematic_behavioural_clamp.png)** | Macro rewired so PAD is driven solely through the Verilog-A `avalanche_gen` element — isolates the behavioral clamp's I–V for direct characterization. |
| **[HBM testbench](Schematics/HBM_testbench_schematic.png)** | 100 pF / 1.5 kΩ human-body-model discharge network (pulse source + R + C) applied to the PAD node of the same behavioral clamp. |

## Results

| File | Description |
|------|-------------|
| **[PAD voltage waveform](Results/GGNMOS_ESD_Pad_Voltage_Waveform.jpg)** | Pad-node response to an 8 V, 100 ps-rise transient stress pulse. |
| **[Spectre run log](Results/ESD_run_log_8V.txt)** | Full transient simulation log — clean run, zero errors. |
| **[Snapback I–V sweep](Results/GGNMOS_snapback_IV_sweep.png)** | Compact-model pad voltage vs. forced current, 1 nA → 100 mA. |
| **[Snapback log](Results/GGNMOS_ESD_snapback_log.txt)** | Snapback sweep simulation log. |
| **[Macro-model I–V sweep](Results/GGNMOS_macro_IV_cadence.png)** | Macro-model characteristic showing the parasitic-NPN trigger knee at ≈ 9 V. |
| **[Macro-model data](Results/macro_IV_data.txt)** | Extracted I–V data points. |
| **[Macro-model Spectre log](Results/macro_spectre.log)** | Full macro-model simulation log. |
| **[Verilog-A snapback I–V](Results/VerilogA_snapback_IV.png)** | Full negative-resistance fold-back — Vt1 = 9.0 V @ 3.98 mA, folding to Vh ≈ 5.35 V @ 20.4 mA. |
| **[Verilog-A netlist](Results/netlist_snapback.txt)** | Spectre netlist confirming the piecewise `V(I)` clamp element and DC sweep setup. |
| **[HBM transient waveform](Results/HBM_Transient.png)** | V(PAD) under a 2 kV HBM pulse — peak 30.31 V @ 16.0 ns, holding shoulder 5.32 V @ 161.1 ns. |
| **[HBM netlist](Results/hbm_netlist.txt)** | Transient netlist — 100 pF/1.5 kΩ HBM network with the convergence settings needed to solve through the clamp's sharp trigger. |

---

## Snapback characterization — the finding

Snapback is the defining behavior of a GGNMOS clamp: at the trigger voltage **Vt1**, avalanche breakdown at the drain junction turns on the device's parasitic NPN, and the voltage *drops* while current climbs — a negative-resistance region that shunts ESD energy at low voltage.

**Method.** The pad was driven with a **current-forced DC sweep** (1 nA → 100 mA, logarithmic) rather than a voltage sweep — in the snapback region multiple currents map to a single voltage, so only forcing current keeps the curve traceable. This is the same principle as TLP characterization on real ESD hardware.

| Ipad | 1 nA | 1 µA | 1 mA | 10 mA | 100 mA |
|---|---|---|---|---|---|
| **V(pad)** | 2.35 V | 6.05 V | 10.65 V | 14.02 V | 25.25 V |

**Finding — the compact model does not snap back.** Pad voltage rises monotonically to 25 V on a 2 V-rated device — physically impossible in real silicon. Spectre even flagged mid-sweep that `Vgd has exceeded the oxide breakdown voltage`, yet the model kept conducting.

**Root cause — a modeling limitation, not a design or tool defect.** gpdk090's BSIM compact models describe normal MOSFET operation but contain no parasitic-bipolar physics: no avalanche-generated substrate current, no base–emitter forward biasing, no negative-resistance region. No process corner or simulator version adds physics the model card does not implement. This is exactly why production ESD flows use **dedicated ESD device models** rather than standard compact models.

---

## Behavioral macro-model — recovering the missing physics

Rather than stop at the limitation, I built a **behavioral macro-model** to reproduce the mechanism the compact model leaves out. The GGNMOS was augmented with:

- an **explicit parasitic NPN** (collector → PAD, emitter → GND, base → internal substrate node),
- a **substrate resistance Rsub** from that node to ground, and
- an **avalanche current source** injecting current from the pad into the substrate node as a function of pad voltage.

Physically, this is the real trigger loop: avalanche current lifts the substrate potential through Rsub, forward-biases the NPN's base, and turns the bipolar on. The avalanche gain (`gav`) and `Rsub` were tuned to bring the loop into conduction.

**Result — the parasitic bipolar triggers.** The macro-model produces a sharp turn-on knee absent from the raw compact model:

| Ipad | 1 µA | 100 µA | 1 mA | 3.98 mA | 100 mA |
|---|---|---|---|---|---|
| **V(pad)** | 0.01 V | 0.69 V | 1.64 V | **9.0 V** | ~23 V |

The device stays off until ~4 mA, then the parasitic NPN turns on at a well-defined **trigger knee Vt1 ≈ 9 V** — behavior the gpdk090 compact model cannot produce at all. Final tuning: `gav = 100 µ`, `Rsub = 10 k`.

**Honest scope at this stage.** This macro-model captures the *trigger* — the bipolar turn-on the compact model omits. It does **not** fold back: every element in the loop (the avalanche source and the parasitic NPN) has positive differential resistance, so no amount of parameter tuning can produce negative resistance. That limit — proven empirically across multiple tuning attempts — is exactly what motivated the next stage.

---

## Verilog-A avalanche source — full snapback recovered

The macro-model's structural limit called for a different approach: instead of trying to make negative resistance *emerge* from device-level elements, the Verilog-A clamp *specifies* the full snapback characteristic directly as a piecewise, current-driven voltage source:
V(I) = Vt1 · (I / It1)                                              for I < It1
V(I) = Vh + (Vt1 − Vh) · exp(−(I − It1)/Ifold) + Ron · (I − It1)     for I ≥ It1

This form was chosen deliberately: it is **single-valued as a function of current**, so under the project's current-forced DC sweep it converges cleanly with no homotopy or gmin-stepping required — the fold-back falls straight out of the equation.

**Extracted from the swept I–V curve:**

| Quantity | Value |
|---|---|
| Trigger point (Vt1, It1) | **9.0 V @ 3.98 mA** |
| Holding point (Vh) | **5.35 V @ 20.4 mA** |

The curve rises linearly to the trigger point, **folds back** as current continues to climb — voltage *dropping* while current rises, the genuine negative-resistance signature of snapback — then resumes rising along a fixed 20 Ω on-resistance. This closes the gap the macro-model could describe but not reach.

**Honest scope.** This is a *behavioral* specification of snapback, not an emergent one — the same approach used in system-level ESD verification, where physical device models aren't available either. Reaching a fully emergent fold-back would require modeling high-current bipolar beta collapse explicitly, a natural extension beyond this project's current scope.

---

## HBM 2 kV stress test — transient verification

With the clamp's DC snapback characterized, the same clamp (unchanged parameters: Vt1 = 9.0 V, Vh ≈ 5.35 V) was stressed with a **2 kV Human Body Model pulse** — the industry-standard 100 pF / 1.5 kΩ discharge network — to verify behavior under a realistic sub-microsecond transient rather than a slow DC sweep.

**Setup.** A 2000 V pulse (10 ns rise, 150 ns fall) drives a 1.5 kΩ series resistor into a 100 pF pad capacitance, applied directly to the clamp's PAD node. Transient analysis ran the full 500 ns event.

**Result:**

| Marker | Time | V(PAD) |
|---|---|---|
| Peak overshoot | 16.0 ns | **30.31 V** |
| Holding shoulder | 161.1 ns | **5.32 V** |

The pad rises fast, overshoots, then decays through a clear holding shoulder at **5.32 V — matching the DC-extracted Vh (5.35 V) almost exactly**, confirming the clamp engages consistently in both the DC and transient domains with no re-tuning.

**Honest scope.** The pad briefly exceeds gate-oxide breakdown before the clamp brings it down. This reflects a genuine limit of a behavioral V(I) model: it cannot reproduce the fast, hard current-sinking of physical bipolar snapback fast enough to fully arrest a sub-nanosecond HBM edge. True HBM survival verification requires the physical device — layout, extracted parasitics, and ultimately silicon — the natural next step.

**Convergence note.** The sharp behavioral transition at trigger caused the transient solver to fail mid-run (`Top 10 Residue too large`) under default settings. Resolved with `cmin = 1f` (tiny node-to-ground capacitance), `method = gear2`, and `maxstep = 1n` — a deliberate numerical fix, not a lucky default.

---

## What this project demonstrates

- **Device design** — built the GGNMOS with the gate permanently grounded, the defining feature behind how the clamp conducts during an ESD event.
- **Stimulus design** — transient testbenches with fast-rise pulses (100 ps ESD pulse, 10 ns HBM edge), because transient behavior — not a DC sweep — governs ESD survival.
- **Characterization methodology** — applied current-forced sweeps to probe snapback, the correct technique for negative-resistance regions.
- **Model-limit analysis** — designed experiments whose outcomes exposed exactly what the PDK's compact models can and cannot represent, twice: once for the compact model, once for the macro-model's structural limits.
- **Behavioral modeling** — built a parasitic-NPN macro-model, then a Verilog-A clamp with a deliberately single-valued `V(I)` formulation, to recover trigger and full fold-back respectively.
- **Numerical debugging** — diagnosed and resolved a transient convergence failure at the clamp's sharp trigger transition using `cmin`, `gear2`, and step-size control.
- **Toolchain debugging** — isolated and fixed a real defect inside the foundry PDK itself, and separately diagnosed a silent CDF-parameter-override issue that was masking Verilog-A edits (below).

---

## Debugging highlights

**PDK resistor model duplication.** Midway through bring-up, Spectre threw repeated `Model 'X' has already been defined` errors originating **inside the PDK's own resistor model file** — not the design.

Root cause: the foundry's `gpdk090.scs` includes the same resistor sub-model once per process corner (NN, SS, SF, FS, FF, each with a high-performance variant — ten includes total). This Spectre build parsed every corner's include instead of scoping to the selected corner, creating duplicate definitions and aborting the run.

Fix: extracted a single clean process-corner section into a standalone model file and repointed the simulator's Model Library Setup at it — restoring a clean, single-definition environment **without touching the device design.**

**Silent CDF override.** A Verilog-A parameter edited directly in the module source appeared to have no effect on simulation results across several re-runs. Tracing it back through the instance's CDF properties (`Edit Object Properties → veriloga` filter) revealed the instance carried its own cached parameter values that silently overrode the module's defaults every time it netlisted. Fix: edited the values on the instance directly rather than the source, and confirmed via the generated netlist — the actual ground truth for what simulates — rather than trusting the GUI dialog.

Isolating both of these meant reading PDK internals and netlist output directly, reasoning about how the netlister, CDF cache, and simulator interact — the kind of unscripted problem-solving that separates *ran a tutorial* from *made unfamiliar tools actually work*.

---

## Roadmap

- [x] **Snapback characterization** — completed; compact model shows no snapback, limitation documented above.
- [x] **Behavioral macro-model** — completed; explicit parasitic NPN recovers the bipolar trigger (Vt1 ≈ 9 V).
- [x] **Verilog-A avalanche source** — completed; piecewise `V(I)` clamp captures full negative-resistance fold-back, Vt1 = 9.0 V / Vh = 5.35 V extracted.
- [x] **HBM stress test** — completed; 2 kV, 100 pF / 1.5 kΩ transient verified against the same DC-characterized clamp.
- [ ] **Layout + DRC / LVS** *(future work)* — physical implementation with drain ballasting.
- [ ] **Multi-finger scaling** *(future work)* — current uniformity and per-finger triggering.

---

<sub>PDK model files are proprietary to the foundry and intentionally excluded from this repository. Only original design work — schematics, testbenches, and results — is included.</sub>
