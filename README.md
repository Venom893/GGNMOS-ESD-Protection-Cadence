# GGNMOS-ESD-Protection-Cadence

![Cadence](https://img.shields.io/badge/Cadence-Virtuoso%20IC6.1.7-E2231A?style=flat-square)
![Spectre](https://img.shields.io/badge/Simulator-Spectre-1f6feb?style=flat-square)
![PDK](https://img.shields.io/badge/PDK-gpdk090%20·%2090nm-2ea043?style=flat-square)
![Domain](https://img.shields.io/badge/Domain-ESD%20Protection%20%2F%20Analog%20I·O-8957e5?style=flat-square)

> Every chip needs ESD protection, or it doesn't survive its first handshake with reality.

A *Gate-Grounded NMOS (GGNMOS)* primary ESD clamp — designed, simulated, and debugged from a blank schematic in Cadence Virtuoso, including a real PDK toolchain bug found and fixed along the way.

---

## Why this matters

Before a chip is ever powered on, it gets handled, shipped, and assembled — and every one of those steps can dump thousands of volts of static onto a pin in nanoseconds. Without an on-die clamp, that energy walks straight into the gate oxide and the chip is dead on arrival.

The GGNMOS is the *workhorse primary ESD clamp* used on real I/O pads in production silicon. It sits quietly on the pad during normal operation and turns into a high-current shunt to ground the instant a discharge hits. Understanding and building one is foundational analog/I-O design work — which is exactly why this project starts there.

---

## At a glance

| | |
|---|---|
| *Tool* | Cadence Virtuoso IC6.1.7 |
| *Simulator* | Spectre Circuit Simulator |
| *PDK / Node* | gpdk090 — 90 nm |
| *Analysis* | Transient (ESD-style stress pulse) |
| *Device* | Gate-Grounded NMOS primary clamp |
| *Status* | Functional — verified clean at 8 V; snapback sweep planned |

---

## Design

| File | Description |
|------|-------------|
| *[GGNMOS cell](Schematics/GGNMOS_ESD_schematic.png)* | NMOS with gate, source, and bulk tied to ground — the configuration that arms the parasitic conduction path during a discharge while keeping the device off in normal use. |
| *[Transient testbench](Schematics/GGNMOS_ESD_tb_schematic.png)* | Pulse source driving the device through the PAD node, emulating a fast-rising ESD-like stress event. |

## Results

| File | Description |
|------|-------------|
| *[PAD voltage waveform](Results/GGNMOS_ESD_Pad_Voltage_Waveform.jpg)* | Pad-node response to an 8 V, 100 ps-rise transient stress pulse. |
| *[Spectre run log](Results/ESD_run_log_8V.txt)* | Full simulation log — clean run, zero errors. |

The waveform shows clean pad-voltage tracking through the protection path at this stress level, confirming a correctly assembled device, testbench, and simulation flow. Pushing the device past its trigger voltage to capture full snapback clamping is the lead item on the roadmap below.

---

## What this project demonstrates

- *Device design* — built the GGNMOS structure with the gate permanently grounded, the defining feature behind how the clamp actually conducts during an ESD event.
- *Stimulus design* — built a transient testbench using a fast-rise (100 ps) pulse, because transient behavior — not a DC sweep — is what governs ESD survival.
- *Verification* — drove a Spectre transient analysis to a clean finish and extracted the pad-voltage response.
- *Toolchain debugging* — isolated and fixed a real defect inside the foundry PDK itself (below), not just the design.

---

## Debugging highlight

Midway through bring-up, Spectre threw repeated Model 'X' has already been defined errors originating *inside the PDK's own resistor model file* — not the design.

*Root cause:* the foundry's gpdk090.scs includes the same resistor sub-model once per process corner (NN, SS, SF, FS, FF, each with a high-performance variant — ten includes total). This Spectre build parsed every corner's include instead of scoping to the selected corner, creating duplicate definitions and aborting the run.

*Fix:* extracted a single clean process-corner section into a standalone model file and repointed the simulator's Model Library Setup at it — restoring a clean, single-definition environment *without touching the device design.*

Isolating this meant reading PDK internals and reasoning about how the netlister and simulator consume corner sections — the kind of unscripted problem-solving that separates ran a tutorial from made unfamiliar tools actually work.

---

## Roadmap

- [ ] *Snapback characterization* — sweep stress voltage (8 V → 20 V) to extract trigger voltage *Vt1* and holding voltage *Vh*, and capture the snapback clamp directly in the transient response.
- [ ] *TLP-style I–V curve* — reconstruct the device's high-current I–V signature to evaluate clamp robustness.
- [ ] *Physical implementation* — draw the layout and run *DRC / LVS* toward tapeout readiness.
- [ ] *Multi-finger scaling* — extend to a fingered device to study current uniformity and per-finger triggering.

---

<sub>PDK model files are proprietary to the foundry and intentionally excluded from this repository. Only original design work — schematics, testbench, and results — is included.</sub>
