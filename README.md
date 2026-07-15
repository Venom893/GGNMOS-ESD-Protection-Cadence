# GGNMOS ESD Protection Cell — Cadence Virtuoso (gpdk090)



![Cadence](https://img.shields.io/badge/Cadence-Virtuoso%20IC6.1.7-E2231A?style=flat-square)




![Spectre](https://img.shields.io/badge/Simulator-Spectre-1f6feb?style=flat-square)




![PDK](https://img.shields.io/badge/PDK-gpdk090%20·%2090nm-2ea043?style=flat-square)




![Domain](https://img.shields.io/badge/Domain-ESD%20Protection%20%2F%20Analog%20I·O-8957e5?style=flat-square)



> Every chip needs ESD protection, or it doesn't survive its first handshake with reality.

A **Gate-Grounded NMOS (GGNMOS)** primary ESD clamp — designed, simulated, characterized, and debugged from a blank schematic in Cadence Virtuoso. Along the way: a real PDK bug found and fixed, a PDK modeling limitation exposed by experiment, and a behavioral macro-model built to recover the physics the compact model leaves out.

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
| **Analyses** | Transient (ESD-style pulse) · DC (current-forced snapback sweep) |
| **Device** | Gate-Grounded NMOS primary clamp + behavioral macro-model |
| **Status** | Transient verified · snapback characterized · macro-model recovers parasitic-NPN trigger (Vt1 ≈ 9 V) |

---

## Design

| File | Description |
|------|-------------|
| **[GGNMOS cell](Schematics/GGNMOS_ESD_schematic.png)** | NMOS with gate, source, and bulk tied to ground — the configuration that arms the parasitic conduction path during a discharge while keeping the device off in normal use. |
| **[Transient testbench](Schematics/GGNMOS_ESD_tb_schematic.png)** | Pulse source driving the device through the PAD node, emulating a fast-rising ESD-like stress event. |
| **[Snapback testbench](Schematics/GGNMOS_ESD_snapback_tb_schematic.png)** | DC current source forcing current into the PAD node — the correct stimulus for tracing a snapback I–V curve. |
| **[Macro-model cell](Schematics/GGNMOS_ESD_macro_schematic.png)** | GGNMOS with an **explicit parasitic NPN + substrate resistance (Rsub) + avalanche current source** — a behavioral model of the bipolar mechanism the compact model omits. |
| **[Macro-model testbench](Schematics/GGNMOS_ESD_macro_tb_schematic.png)** | Current-forced testbench driving the macro-model to extract its trigger characteristic. |

## Results

| File | Description |
|------|-------------|
| **[PAD voltage waveform](Results/GGNMOS_ESD_Pad_Voltage_Waveform.jpg)** | Pad-node response to an 8 V, 100 ps-rise transient stress pulse. |
| **[Spectre run log](Results/ESD_run_log_8V.txt)** | Full transient simulation log — clean run, zero errors. |
| **[Snapback I–V sweep](Results/GGNMOS_snapback_IV_sweep.png)** | Compact-model pad voltage vs. forced current, 1 nA → 100 mA. |
| **[Macro-model I–V sweep](Results/GGNMOS_macro_IV_cadence.png)** | Macro-model characteristic showing the parasitic-NPN trigger knee at ≈ 9 V. |
| **[Macro-model data](Results/macro_IV_data.txt)** | Extracted I–V data points. |
| **[Macro-model Spectre log](Results/macro_spectre.log)** | Full macro-model simulation log. |

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

**Honest scope.** This macro-model captures the *trigger* — the bipolar turn-on the compact model omits. Full negative-resistance *fold-back* (voltage dropping to a holding level Vh) requires the avalanche generation to be a nonlinear, voltage-dependent term, best implemented as a **Verilog-A behavioral source** — the planned next step below.

---

## What this project demonstrates

- **Device design** — built the GGNMOS with the gate permanently grounded, the defining feature behind how the clamp conducts during an ESD event.
- **Stimulus design** — transient testbench with a fast-rise (100 ps) pulse, because transient behavior — not a DC sweep — governs ESD survival.
- **Characterization methodology** — applied a current-forced sweep to probe snapback, the correct technique for negative-resistance regions.
- **Model-limit analysis** — designed an experiment whose outcome exposed exactly what the PDK's compact models can and cannot represent.
- **Behavioral modeling** — built a parasitic-NPN macro-model and tuned its avalanche feedback to recover the bipolar trigger the compact model omits.
- **Toolchain debugging** — isolated and fixed a real defect inside the foundry PDK itself (below), not just the design.

---

## Debugging highlight

Midway through bring-up, Spectre threw repeated `Model 'X' has already been defined` errors originating **inside the PDK's own resistor model file** — not the design.

**Root cause:** the foundry's `gpdk090.scs` includes the same resistor sub-model once per process corner (NN, SS, SF, FS, FF, each with a high-performance variant — ten includes total). This Spectre build parsed every corner's include instead of scoping to the selected corner, creating duplicate definitions and aborting the run.

**Fix:** extracted a single clean process-corner section into a standalone model file and repointed the simulator's Model Library Setup at it — restoring a clean, single-definition environment **without touching the device design.**

Isolating this meant reading PDK internals and reasoning about how the netlister and simulator consume corner sections — the kind of unscripted problem-solving that separates *ran a tutorial* from *made unfamiliar tools actually work*.

---

## Roadmap

- [x] **Snapback characterization** — completed; compact model shows no snapback, limitation documented above.
- [x] **Behavioral macro-model** — completed; explicit parasitic NPN recovers the bipolar trigger (Vt1 ≈ 9 V).
- [ ] **Verilog-A avalanche source** — replace the linear generator with a voltage-dependent (exponential) avalanche term to capture full negative-resistance fold-back and extract the holding voltage Vh.
- [ ] **HBM stress test** — 100 pF / 1.5 kΩ industry-standard discharge network.
- [ ] **Layout + DRC / LVS** — physical implementation with drain ballasting.
- [ ] **Multi-finger scaling** — current uniformity and per-finger triggering.

---

<sub>PDK model files are proprietary to the foundry and intentionally excluded from this repository. Only original design work — schematics, testbenches, and results — is included.</sub>
