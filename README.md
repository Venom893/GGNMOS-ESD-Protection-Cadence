# GGNMOS ESD Protection Cell — Cadence Virtuoso (gpdk090)



![Cadence](https://img.shields.io/badge/Cadence-Virtuoso%20IC6.1.7-E2231A?style=flat-square)




![Spectre](https://img.shields.io/badge/Simulator-Spectre-1f6feb?style=flat-square)




![PDK](https://img.shields.io/badge/PDK-gpdk090%20·%2090nm-2ea043?style=flat-square)




![Domain](https://img.shields.io/badge/Domain-ESD%20Protection%20%2F%20Analog%20I·O-8957e5?style=flat-square)



> Every chip needs ESD protection, or it doesn't survive its first handshake with reality.

A **Gate-Grounded NMOS (GGNMOS)** primary ESD clamp — designed, simulated, characterized, and debugged from a blank schematic in Cadence Virtuoso, including a real PDK toolchain bug found and fixed, and a PDK modeling limitation identified through direct experiment.

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
| **Analyses** | Transient (ESD-style stress pulse) · DC (current-forced snapback sweep) |
| **Device** | Gate-Grounded NMOS primary clamp |
| **Status** | Verified clean at 8 V transient · snapback characterized over 8 decades — PDK model limitation identified |

---

## Design

| File | Description |
|------|-------------|
| **[GGNMOS cell](Schematics/GGNMOS_ESD_schematic.png)** | NMOS with gate, source, and bulk tied to ground — the configuration that arms the parasitic conduction path during a discharge while keeping the device off in normal use. |
| **[Transient testbench](Schematics/GGNMOS_ESD_tb_schematic.png)** | Pulse source driving the device through the PAD node, emulating a fast-rising ESD-like stress event. |
| **[Snapback testbench](Schematics/GGNMOS_ESD_snapback_tb_schematic.png)** | DC current source forcing current into the PAD node — the correct stimulus for tracing a snapback I–V curve. |

## Results

| File | Description |
|------|-------------|
| **[PAD voltage waveform](Results/GGNMOS_ESD_Pad_Voltage_Waveform.jpg)** | Pad-node response to an 8 V, 100 ps-rise transient stress pulse. |
| **[Spectre run log](Results/ESD_run_log_8V.txt)** | Full transient simulation log — clean run, zero errors. |
| **[Snapback I–V sweep](Results/GGNMOS_snapback_IV_sweep.png)** | Pad voltage vs. forced pad current, 1 nA → 100 mA, with extracted data markers. |
| **[Snapback Spectre log](Results/snapback_spectre.log)** | Full DC sweep log, including the simulator's oxide-breakdown warning. |

The transient waveform shows clean pad-voltage tracking through the protection path at this stress level, confirming a correctly assembled device, testbench, and simulation flow.

---

## Snapback characterization

Snapback is the defining behavior of a GGNMOS clamp: at the trigger voltage **Vt1**, avalanche breakdown at the drain junction turns on the device's parasitic NPN bipolar, and the voltage *drops* while current climbs — a negative-differential-resistance region that shunts ESD energy at low voltage.

**Method.** The pad was driven with a **current-forced DC sweep** (1 nA → 100 mA, logarithmic, 20 points/decade) rather than a voltage sweep. This choice matters: in the snapback region multiple currents map to a single voltage, so a voltage sweep cannot trace the curve — forcing current and measuring voltage keeps it single-valued. This is the same principle behind TLP (Transmission Line Pulse) characterization used on real ESD hardware.

**Measured response:**

| Forced pad current | Pad voltage |
|---|---|
| 1 nA | 2.35 V |
| 1 µA | 6.05 V |
| 1 mA | 10.65 V |
| 10 mA | 14.02 V |
| 100 mA | 25.25 V |

**Finding — no snapback occurs.** The pad voltage rises monotonically across eight decades of current, reaching 25 V at 100 mA on a 2 V-rated device — behavior that is physically impossible in real silicon. Spectre itself flagged the contradiction mid-sweep, warning that `Vgd has exceeded the oxide breakdown voltage`, yet the model kept conducting.

**Root cause — a modeling limitation, not a design or tool defect.** gpdk090's BSIM compact models describe normal MOSFET operation (on/off, saturation, leakage, junction conduction) but contain no parasitic-bipolar physics: no avalanche-generated substrate current, no base–emitter forward biasing, no negative-resistance region. No process corner or simulator version adds physics the model card does not implement.

This is exactly why production ESD flows rely on **dedicated ESD device models** supplied by the foundry — or hand-built macro-models (MOSFET + explicit parasitic NPN + substrate resistance + avalanche current source) — rather than standard compact models. Reaching the edge of the PDK, recognizing it, and identifying the industry-standard path past it is the core result of this experiment.

---

## What this project demonstrates

- **Device design** — built the GGNMOS structure with the gate permanently grounded, the defining feature behind how the clamp actually conducts during an ESD event.
- **Stimulus design** — built a transient testbench using a fast-rise (100 ps) pulse, because transient behavior — not a DC sweep — is what governs ESD survival.
- **Characterization methodology** — applied a current-forced sweep to probe snapback, the correct technique for negative-resistance regions, and extracted a full I–V dataset across eight decades.
- **Verification** — drove Spectre transient and DC analyses to clean finishes and extracted pad-voltage responses.
- **Toolchain debugging** — isolated and fixed a real defect inside the foundry PDK itself (below), not just the design.
- **Model-limit analysis** — designed an experiment whose outcome exposed what the PDK's compact models can and cannot represent, and mapped the finding to industry practice.

---

## Debugging highlight

Midway through bring-up, Spectre threw repeated `Model 'X' has already been defined` errors originating **inside the PDK's own resistor model file** — not the design.

**Root cause:** the foundry's `gpdk090.scs` includes the same resistor sub-model once per process corner (NN, SS, SF, FS, FF, each with a high-performance variant — ten includes total). This Spectre build parsed every corner's include instead of scoping to the selected corner, creating duplicate definitions and aborting the run.

**Fix:** extracted a single clean process-corner section into a standalone model file and repointed the simulator's Model Library Setup at it — restoring a clean, single-definition environment **without touching the device design.**

Isolating this meant reading PDK internals and reasoning about how the netlister and simulator consume corner sections — the kind of unscripted problem-solving that separates *ran a tutorial* from *made unfamiliar tools actually work*.

---

## Roadmap

- [x] **Snapback characterization** — current-forced I–V sweep completed (1 nA → 100 mA); no snapback observed, gpdk090 model limitation identified and documented above.
- [ ] **Behavioral macro-model** — assemble MOSFET + explicit parasitic NPN + substrate resistance + avalanche source to recover the snapback physics the compact model omits, and extract Vt1 / Vh from it.
- [ ] **HBM-style stress test** — 100 pF / 1.5 kΩ discharge network for industry-standard Human Body Model qualification stress.
- [ ] **Physical implementation** — draw the layout with drain ballasting and run **DRC / LVS** toward tapeout readiness.
- [ ] **Multi-finger scaling** — extend to a fingered device to study current uniformity and per-finger triggering.

---

<sub>PDK model files are proprietary to the foundry and intentionally excluded from this repository. Only original design work — schematics, testbench, and results — is included.</sub>
