# Automated Peristaltic Pump Dispensing System
### PLC-HMI Precision Liquid Dosing | MEPL, Kathmandu

> **Status:** Complete — deployed for commissioning  
> **Role:** Sole designer and programmer  
> **Firm:** Mechatronics Engineering Pvt. Ltd. (MEPL)  
> **Date:** May 2026

---

## Overview

An automated precision liquid dispensing system designed for low-volume medical and laboratory applications — specifically ophthalmic solution (eye-drop) measurement and dosing. The system replaces manual dispensing, which suffers from human error, inconsistent flow rates, and poor repeatability.

The operator enters a target volume on the HMI. The PLC converts that into an exact pulse count, drives a stepper motor through a peristaltic pump, and stops automatically when the target is reached — accurate to ±0.1 mL across the full operating range.

**Key highlights:**
- Validated dispensing accuracy of ±0.1 mL across 5–20 mL range
- Experimental K-factor calibration engine — no fluid dynamics calculations required
- Self-updating calibration routine operable by non-technical staff
- Four engineering challenges encountered and resolved during development
- Foot switch integration for hands-free operation

---

## Hardware

| S.N | Component | Qty |
|-----|-----------|-----|
| 1 | Peristaltic Pump | 1 |
| 2 | Coolmay PLC-HMI (Mitsubishi FX3U-compatible architecture) | 1 |
| 3 | DP508D Stepper Motor Driver | 1 |
| 4 | 24V DC Power Supply | 1 |
| 5 | Foot Controlled Switch | 1 |

> The Coolmay PLC-HMI uses a Mitsubishi FX3U-compatible instruction set and was programmed using GX Works2. All ladder logic instructions, register maps, and high-speed pulse outputs are fully compatible with standard Mitsubishi FX3U architecture.

---

## Calibration Approach

Rather than relying on theoretical fluid dynamics (tube cross-section, roller compression, fluid viscosity), an experimental K-factor approach was adopted. Real-world peristaltic pumps drift from theoretical models due to tubing elasticity, mechanical tolerances, and fluid characteristics — so a measured calibration constant is more reliable.

**Core relationship:**

```
Dispensed Volume = K × Pulse Count
Pulse Count      = Target Volume / K
```

Where **K** is experimentally determined by running a known pulse count and measuring the physical output. This constant is stored in retentive PLC memory (D202) and survives power cycles.

**The calibration routine is operator-driven — no programming required to recalibrate:**
1. Run a test batch at a known target volume
2. Physically measure the actual output with a graduated cylinder
3. Enter the measured value into the HMI
4. Press Calibrate — the PLC recalculates and saves the new K-factor automatically

---

## Operating Modes

### Continuous Mode (Manual / Priming)
For line priming, purging, or indefinite transfer.

The operator sets direction via the global M1 toggle (OFF = clockwise, ON = anticlockwise) and holds the manual run button (M111). The PLC loads a constant 15,000 pulses/sec into D150 and fires `DPLSV`. Releasing the button instantly terminates the pulse train — no trailing or slipping.

### Batch Mode (Target Volume)
For precise, repeatable dosing.

The operator enters a target volume into D100 and presses Auto Start (M112). The PLC calculates:

```
Target Pulses (D206) = Target Volume (D100) / K-Factor (D202)
```

The result is fed as a hard limit into `DPLSY` at 10,000 pulses/sec. The cycle self-terminates the moment the hardware counter matches D206 via execution complete flag M8029.

### Calibration Mode
Operator-driven K-factor update without any programming. Full procedure in the calibration section above. Protected against divide-by-zero via DECMP gate on D120.

---

## Ladder Logic Architecture

**Source file:** `/ladder-logic/peristaltic_pump.gx3`  
**PDF export:** `/ladder-logic/peristaltic_pump_logic.pdf`

### Key Rungs

**Rung 0 — Master Direction Control**  
Bit M1 acts as the sole controller for direction output Y001. Manual and automatic routines both route through M1 rather than independently writing to Y001 — this eliminates double-coil conflicts entirely.

**Rung 20 — System Initialization (M8002 first-scan pulse)**  
On CPU boot, default pulse velocity (D110 = 10,000 Hz), floating-point calibration constants, and initial dosing values are written to data registers. Guarantees a predictable machine state every power-on.

**Rungs 71–91 — Manual & Priming Mode**  
`DPLSV` triggered by M111. Direction sign bit rerouted to unused virtual output Y020 to avoid double-coil conflict with the M1 master direction rung.

**Rungs 174–210 — Automated Dosing Loop**  
Reads D100 (target volume), divides against D202 (K-factor), converts to integer pulse count in D206, fires `DPLSY`. Loop self-terminates on M8029 (execution complete flag).

**Rungs 120–125 — Safe Calibration Gate**  
`DECMP` block checks D120 against E0 (floating-point zero) before any math executes. If D120 = 0.0, M21 trips and blocks the entire calibration branch. Only M22 (valid non-zero input confirmed) opens the execution path.

**Rungs 138+ — K-Factor Recalculation Sequence (M40)**  
When triggered by a valid calibration input:
1. `DFLT` — converts raw pulse integer D206 to 32-bit float D230
2. `DEDIV` — calculates pulse ratio: D232 = theoretical base / actual extrapolated pulses
3. `DEMUL` — new K-factor: D202 = measured volume (D120) × pulse ratio (D232)

New K-factor overwrites D202 in retentive memory — survives power loss.

---

## HMI Design

Designed in mView. Single-screen interface with:
- Target Volume numeric entry (32-bit float → D100)
- Speed (Hz) entry field
- Measured Volume entry for calibration (32-bit float → D120)
- Direction toggle (M1)
- Calibrate button (momentary pulse → M15)
- Emergency Stop (prominent, right side)
- Live parameter display

**Key HMI engineering decisions:**

**Register type alignment:** All numeric fields mapped as 32-bit float to match the PLC's math depth. Mismatching this caused Challenge 2 (see below).

**Momentary pulse actions for command triggers:** Calibrate and Auto Start buttons use momentary (not Set ON) action types. This gives the PLC's RST instruction authority to clean up bits without the HMI overwriting them on the next refresh cycle.

**Volatile vs retentive separation:** Temporary user entries (D120) assigned to volatile registers, cleared on M8002 at every boot. System constants (D202) in retentive memory — persistent across power cycles.

---

## Engineering Challenges & Solutions

Four real integration problems were encountered and resolved during development. Documented here as engineering record.

---

### Challenge 1: Asynchronous HMI–PLC Race Condition (Stuck Calibration Button)

**Problem:** The calibration trigger bit M15 would permanently latch high. The PLC's `RST M15` was executing correctly but the bit remained set, causing infinite re-triggering of the calibration math.

**Root cause:** The HMI button was configured as a persistent "Set ON" action. HMI cyclic communication runs asynchronously to the PLC scan cycle. The sequence: operator presses button → HMI writes 1 to M15 → PLC scans, executes math, resets M15 to 0 → HMI reads 0 on next refresh, interprets it as a state mismatch, forcibly overwrites back to 1. The HMI was winning the race on every cycle.

**Solution:** Button action type changed from "Set ON" to "Momentary pulse" in mView. This strips the HMI of forcing authority — the physical touch sets the bit, the PLC's RST drops it within one scan, and the HMI has no grounds to reinstate it. Race condition eliminated.

---

### Challenge 2: Floating-Point Display Corruption (Scrambled HMI Values)

**Problem:** HMI numeric fields displayed scrambled values, overflow characters, or large random numbers despite PLC registers containing valid data.

**Root cause:** Data type mismatch between the PLC backend and HMI frontend. GX Works2 math blocks (DEDIV, DEMUL, DECMP) were operating on 32-bit IEEE-754 floating-point values. The mView numeric entry fields were configured as `[32Bit] Unsigned Integer` display format. The HMI was interpreting raw IEEE-754 binary encoding as integer packets — resulting in garbage display values.

**Solution:** mView numeric entry data format properties changed to `[32Bit] Float` across all relevant fields. Telemetry alignment restored, decimal precision accurate to 0.1 mL.

---

### Challenge 3: CPU Math Halt Prevention (Divide-by-Zero Gate)

**Problem:** If the operator pressed Calibrate with D120 at 0.0 (empty field), the calibration math would attempt a division by zero — causing an illegal operation fault and tripping the industrial PLC's emergency stop. A CPU halt in a production environment is a serious failure.

**Root cause:** The calibration sequence had no input validation before executing the division block. Standard PLC processors fault-out on divide-by-zero, triggering safety stops.

**Solution:** `DECMP` floating-point comparison block inserted at the absolute front-end of the calibration rung. If D120 equals E0 (floating-point zero), status bit M21 trips and bypasses the entire execution branch. The math only fires when M22 confirms a valid, positive, non-zero input. System is completely foolproof against accidental empty-field calibration triggers.

---

### Challenge 4: Stale Data on Power Cycles (Register Memory Leak)

**Problem:** Temporary calibration entry values persisted in memory after power loss or mid-sequence aborts. On the next boot, stale partial numbers appeared in the HMI entry fields, creating operator confusion and potentially corrupting a calibration sequence.

**Root cause:** Temporary user entry registers were mapped to D250-range addresses — a retentive memory block hardware-latched by the FX3U's internal battery backup. Retentive memory is appropriate for system constants (K-factor), not transient user inputs.

**Solution:** Temporary entry field migrated to D120, a non-retentive volatile register. An initialization rung driven by M8002 (first-scan pulse) overwrites D120 with E0 (floating-point zero) on every cold boot. The calibration constant in D202 (retentive) remains intact. Clean separation: volatile for inputs, retentive for constants.

---

## Testing & Validation

**Dry-run logic validation:** Full system tested in GX Works2 bit monitor before any liquid was introduced. HMI–PLC handshake, direction control, and interlock behavior verified against all edge cases.

**Fault mitigation testing:** Divide-by-zero gate stress-tested by deliberately triggering calibration with empty register. DECMP successfully blocked math execution on every attempt.

**Volumetric calibration — multi-point linearity test:**

| Target Volume | Result |
|---------------|--------|
| 5.0 mL | Within ±0.1 mL |
| 7.5 mL | Within ±0.1 mL |
| 10.0 mL | Baseline calibration point |
| 15.0 mL | Within ±0.1 mL |
| 20.0 mL | Within ±0.1 mL |

Precision held consistent across the full operating range — confirming the floating-point calibration engine eliminates linear scaling errors.

**Cold-boot memory verification:** System power-cycled with stale data in D120. On boot, M8002 initialization confirmed to overwrite D120 with E0 while leaving D202 (K-factor) untouched in retentive memory. Total memory isolation confirmed.

---

## Repository Structure

```
Peristaltic-Pump-System/
    README.md
    /ladder-logic/
        peristaltic_pump.gx3        ← GX Works2 source file
        peristaltic_pump_logic.pdf  ← Exported ladder logic PDF
    /docs/
        Peristaltic_Pump_Report.pdf ← Full technical report
    /Photos/
        (Build phase photos and HMI screenshots)
```

---

## Tools & Technologies

| Domain | Details |
|--------|---------|
| PLC Platform | Coolmay PLC-HMI (Mitsubishi FX3U-compatible) |
| Programming Software | GX Works2 (Mitsubishi Electric) |
| HMI Design | mView |
| Motor Driver | DP508D Stepper Motor Driver |
| Key Instructions | DPLSY, DPLSV, DECMP, DEDIV, DEMUL, DFLT, DINT |
| Memory Architecture | Retentive (D200–D511) for constants, volatile (D120) for inputs |
| Validation Method | Multi-point wet-test with graduated 20 mL laboratory vessel |

---

*Authored by Sambridda Ranabhat | Mechatronics Engineering Pvt. Ltd. (MEPL) | 2026*
