# 8-Bit Segmented Current-Steering DAC (65nm CMOS)

An 8-bit, 3+5 segmented current-steering DAC designed in a 65nm CMOS process, with a Verilog-A thermometer decoder, full static and dynamic verification, and Monte Carlo mismatch analysis.

| | |
| :--- | :--- |
| **Course** | ECE 412s – Analog Integrated Circuit Design, Ain Shams University (Fall 2026) |
| **Team Members** | Adam Elalyly, Ahmed Hossam, Mohammed Wael |
| **Technology** | 65nm CMOS PDK, V<sub>DD</sub> = 1.2 V |
| **Tools** | Cadence Virtuoso, Spectre / APS, ADE Explorer & Assembler, Verilog-A |
| **License** | MIT |

---

## Table of Contents

1. [Overview](#overview)
2. [Results at a Glance](#results-at-a-glance)
3. [Architecture](#architecture)
4. [Sizing and Source Degeneration](#sizing-and-source-degeneration)
5. [Verification Methodology](#verification-methodology)
6. [Repository Structure](#repository-structure)
7. [Getting Started](#getting-started)
8. [License](#license)

---

## Overview

This repository contains the Cadence Virtuoso schematics, the Verilog-A behavioral decoder and testbenches, simulation waveforms, and exported characterization data for the DAC.

To balance linearity (low DNL and low glitch energy at major-carry transitions) against routing and decoder complexity, the design uses a **3+5 hybrid segmentation**:

- **3 MSBs, unary:** a 3-to-7 binary-to-thermometer decoder drives **7 identical slices**, each carrying 32 × I<sub>LSB</sub> = 64 µA.
- **5 LSBs, binary:** driven directly into **5 binary-weighted slices** of 16, 8, 4, 2 and 1 × I<sub>LSB</sub>.
- **Cascoded current sources:** every slice stacks a PMOS cascode device (M<sub>CAS</sub>) on the unit current source (M<sub>CS</sub>) to raise output impedance and reduce the parabolic INL bow across the output swing.
- **Skewed switch drive:** differential steering switches use skewed crossing points to limit common-source voltage spikes and glitches.

---

## Results at a Glance

| Parameter | Target | Achieved |
| :--- | :--- | :--- |
| Resolution | 8 bits (codes 0 to 255) | 8 bits |
| Supply voltage | 1.2 V | 1.2 V |
| Reference current (I<sub>ref</sub>) | 10 µA, single input | 10 µA, giving I<sub>LSB</sub> = 2 µA |
| Single-ended output swing | 0 to 700 mV | 0 to 700 mV (V<sub>LSB</sub> ≈ 2.745 mV) |
| INL (±3σ Monte Carlo) | within ±0.7 LSB | within ±0.7 LSB |
| DNL (±3σ Monte Carlo) | within ±0.7 LSB | within ±0.7 LSB |
| ENOB (f<sub>in</sub> = 19.53125 kHz, f<sub>s</sub> = 1 MHz) | ≥ 7.5 bits | **7.81 bits** (256-point coherent FFT) |

---

## Architecture

### Key parameters

| Quantity | Value | Derivation |
| :--- | :--- | :--- |
| Unit LSB current, I<sub>LSB</sub> | 2 µA | 10 µA reference through a 5:1 mirror (unit slice M = 1 carries 2 µA; reference branch M = 5 carries 10 µA) |
| Full-scale current, I<sub>FS</sub> | 510 µA | 255 × 2 µA (7 × 32 + 31 = 255 LSB) |
| Load resistors, R<sub>pos</sub> = R<sub>neg</sub> | 1.3725 kΩ | 700 mV / 510 µA |
| LSB voltage, V<sub>LSB</sub> | ≈ 2.745 mV | 700 mV / 255 |

### Steering switches

65nm PMOS differential pairs (W = 2.0 µm, L = 60 nm, bulk tied to V<sub>DD</sub>), driven with skewed crossing points.

---

## Sizing and Source Degeneration

The array has **12 signal slices (7 unary + 5 binary) plus one diode-connected reference branch**. Source degeneration resistors (R<sub>s</sub> = 0.1 V / I<sub>branch</sub>) sit in series with V<sub>DD</sub> at each PMOS source, to evaluate how much they suppress ±3σ mismatch.

| Slice | Control | M | Branch current | R<sub>s</sub> (0.1 V drop) |
| :--- | :--- | :---: | :---: | :---: |
| Reference | Diode-connected | 5 | 10 µA | 10.000 kΩ |
| Unary 1–7 | T<sub>0</sub>–T<sub>6</sub> / T̄<sub>0</sub>–T̄<sub>6</sub> | 32 each | 64 µA each | 1.5625 kΩ each |
| Binary B<sub>4</sub> | B<sub>4</sub> / B̄<sub>4</sub> | 16 | 32 µA | 3.125 kΩ |
| Binary B<sub>3</sub> | B<sub>3</sub> / B̄<sub>3</sub> | 8 | 16 µA | 6.250 kΩ |
| Binary B<sub>2</sub> | B<sub>2</sub> / B̄<sub>2</sub> | 4 | 8 µA | 12.50 kΩ |
| Binary B<sub>1</sub> | B<sub>1</sub> / B̄<sub>1</sub> | 2 | 4 µA | 25.00 kΩ |
| Binary B<sub>0</sub> (LSB) | B<sub>0</sub> / B̄<sub>0</sub> | 1 | 2 µA | 50.00 kΩ |

---

## Verification Methodology

1. **DC operating point and headroom (`dcOp`).** Confirms the endpoint voltages (0 mV at code 0, 700 mV at code 255), the major-carry step (≈ 2.745 mV), and that every M<sub>CS</sub> and M<sub>CAS</sub> device is in saturation (`region = 2`).
2. **Full-code ramp transient (static linearity).** Sweeps all 256 codes at 1 µs per step (256 µs total) so each code settles fully before sampling. Endpoint-fit **DNL** and **INL** are computed from this sweep.
3. **Coherent sine FFT (dynamic linearity).** A quantized sine at f<sub>test</sub> = (5/256) × 1 MHz = 19.53125 kHz with f<sub>s</sub> = 1 MHz is analyzed with a 256-point rectangular-window FFT (25.6 µs to 281.6 µs), giving **ENOB = 7.81 bits**.
4. **Monte Carlo mismatch (±3σ).** Mismatch-only Monte Carlo over the 256-code ramp, run both **before** and **after** adding the 0.1 V source degeneration resistors, to quantify the reduction in INL/DNL spread.

---

## Repository Structure

```text
├── DAC/                      # Virtuoso library: schematics, symbols, Verilog-A views
├── cds.lib                   # Cadence library path configuration
├── SaveOP.scs                # Spectre operating-point save script (device region checks)
├── 1.csv, 2.csv, HOSS.csv    # Exported ViVA waveforms (DNL, INL, ramp and FFT data)
├── dac.png, unary.png, ...   # Schematic captures, testbench setups, linearity/FFT plots
├── LICENSE                   # MIT License
└── README.md                 # This file
```

---

## Getting Started

1. Clone the repository and `cd` into it.
2. Launch Virtuoso from the repository root so `cds.lib` is picked up, and make sure it points to your 65nm PDK.
3. Open the `DAC` library and choose the testbench for the analysis you want (DC operating point, ramp, FFT, or Monte Carlo).
4. Run the simulation in ADE Explorer or Assembler with Spectre / APS.
5. Use `SaveOP.scs` to dump operating points when checking device saturation.

> The PDK is not included. You need your own licensed access to the 65nm process design kit and to Cadence tools.

---

## License

Released under the [MIT License](LICENSE).
