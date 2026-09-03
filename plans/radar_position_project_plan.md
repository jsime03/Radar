# K-LC7 Radar Positioning Project — Plan

## Goal
Build a DIY 2D positioning system (range + angle) using the RFbeam K-LC7 24GHz radar
transceiver, driven and processed entirely by an STM32, starting from motion detection
and incrementally adding range and angle capability.

## Hardware

| Item | Notes |
|---|---|
| RFbeam K-LC7 (THT-type) | 24GHz radar transceiver, 1 Tx / 2 Rx, raw analog I/Q outputs |
| STM32 (F4/F7/H7-class recommended) | Needs hardware FPU + DMA-capable ADC for FFT workload |
| DAC or ramp-generation circuit | Drives VCO_In for FMCW sweep (Phase 3) |
| Breadboard / perfboard + jumper wires | No PCB fab or reflow needed — THT + 2.54mm pitch |

### K-LC7 key specs to remember
- Antenna spacing (Rx1/Rx2): **8.763 mm**
- Wavelength at 24.125GHz: **12.43 mm**
- Beam aperture: 80° (horizontal) × 34° (vertical)
- VCO tuning range: 200–350 MHz, driven by 0–5V on VCO_In pin
- Outputs: Out_I1, Out_Q1 (Rx1), Out_I2, Out_Q2 (Rx2) — all analog, 50Ω, load 1kΩ
- IF frequency range: 0–50 MHz (-3dB bandwidth)
- Supply: 3.2–5.5V, ~90mA

### Pinout (from datasheet)
| Pin | Signal |
|---|---|
| X1,1 | Vcc (+3.2 to +5.5V) |
| X1,2 | GND |
| X1,3 | fDiv_out (VCO freq / 8192, for monitoring only) |
| X2,1 | Out_I1 (Rx1 in-phase) |
| X2,2 | Out_Q1 (Rx1 quadrature) |
| X2,3 | GND |
| X2,4 | Out_Q2 (Rx2 quadrature) |
| X2,5 | Out_I2 (Rx2 in-phase) |
| X2,6 | VCO_In (0–5V analog sweep input) |

---

## Phase 1 — Wiring & Bring-up
- Wire K-LC7 to breadboard, power from 3.3–5V rail (confirm STM32 board can supply
  enough current, ~90mA).
- Leave VCO_In open/floating (internal ~1.2V bias) for fixed-frequency CW mode.
- Connect Out_I1 (and optionally Q1) to STM32 ADC input.
- **Signal conditioning note:** IF outputs are low-level analog — keep wires short,
  consider simple op-amp buffering/filtering ahead of the ADC (also sets up the
  conditioning stage you'll want for Phase 2/3 anyway).

**Milestone:** Confirm you can read a nonzero, stable DC-ish signal on Out_I1 with no
target present, and see it change when you wave a hand in front of the sensor.

---

## Phase 2 — Motion Detection (CW mode)
- Sample Out_I1 (and Q1) continuously via ADC + DMA.
- Detect motion via simple thresholding on signal variance, or a lightweight FFT to
  extract Doppler frequency.
- Use I/Q phase relationship (I1 vs Q1) to determine direction (toward vs. away).

**Milestone:** Reliable "motion detected: yes/no + direction" output, no range yet.

**No new hardware needed** — this stage is pure firmware on top of Phase 1 wiring.

---

## Phase 3 — Ranging (FMCW)
- Generate a linear voltage ramp (0–5V) on VCO_In using STM32 DAC (or external
  ramp circuit) to sweep transmit frequency — this is the FM in FMCW.
- Sample the IF beat signal (Out_I1) during each sweep.
- Run FFT on the sampled sweep — beat frequency maps to range via the standard
  FMCW range equation.
- Tune sweep rate / bandwidth for desired range resolution vs. update rate trade-off.

**Milestone:** Distance-to-target readout (single Rx channel, no angle) — equivalent
capability to an off-the-shelf single-beam range sensor, but self-built.

---

## Phase 4 — Angle / Position (AoA)
- Repeat the Phase 3 range-FFT independently on **both** Rx channels (I1/Q1 and I2/Q2).
- For a detected target at a given range bin, compute phase difference ΔΦ between
  the two channels.
- Apply angle-of-arrival formula (from K-LC7 datasheet):

  ```
  θ = arcsin(0.2257 × ΔΦ)     [ΔΦ in radians]
  ```

  (derived from λ = 12.43mm, antenna spacing l = 8.763mm)

- Combine (range, θ) → 2D polar position of target relative to sensor.
- Note the "uniqueness area" limit from the datasheet's phase-vs-angle curve — ΔΦ
  wraps beyond roughly ±180°, so very large angles become ambiguous.

**Milestone:** Full 2D position output (range + angle) for a single target — project's
core goal achieved with fully custom DSP on the STM32.

---

## Phase 5 — Stretch Goals (optional, pick based on remaining time)
- **Multi-target handling:** peak-picking across multiple range/Doppler bins instead
  of single strongest target.
- **Mechanical scanning:** mount on a stepper/servo for step-stare wide-area coverage
  (accept beamwidth-limited angular resolution; dwell time needed per step since
  Doppler/motion detection depends on sustained observation).
- **Upgrade path:** if resolution/range outgrows the K-LC7, revisit the RFbeam V-LD3
  (TI IWRL6432-based) — but that requires a custom carrier PCB + reflow soldering to
  access raw SPI data, so only worth it once the K-LC7 pipeline is validated and
  there's runway for a PCB fab/assembly cycle.

---

## Known Risks / Things to Watch
- STM32 compute headroom for real-time FFT — validate early with Phase 3, before
  building Phase 4 on top of a struggling pipeline.
- Analog signal integrity on breadboard wiring — noise pickup, especially near
  switching regulators/motors on the bench.
- FMCW ramp linearity — a non-linear VCO_In sweep smears the range-FFT; may need
  DAC output filtering or calibration against known VCO tuning curve.
- Phase calibration between Rx1/Rx2 channels (ADC sampling skew, cable length
  mismatch) will directly bias the angle measurement — worth a simple two-point
  calibration (known target at 0° and at a second known angle) before trusting θ.
