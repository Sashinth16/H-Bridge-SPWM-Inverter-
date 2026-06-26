# Single-Phase H-Bridge SPWM Inverter
**LTSpice Simulation | Infineon SPA11N60C3 CoolMOS | 200V DC → 50Hz AC**

Designed and simulated a full-bridge DC-AC inverter converting a 200V DC bus to a 50Hz sinusoidal AC output using Sinusoidal PWM at 50kHz switching frequency. The project covers the complete inverter design process: MOSFET selection, SPWM generation, dead-time implementation, LC output filtering, harmonic analysis, and efficiency evaluation under resistive and RL loads.

---

## Specifications

| Parameter | Value |
|---|---|
| DC Bus Voltage | 200V |
| Output Voltage | ~111V RMS / 157.4V peak |
| Output Frequency | 50Hz |
| Switching Frequency | 50kHz |
| Modulation Index | 0.85 |
| Dead Time | 300ns |
| LC Filter Cutoff | 5kHz |
| Filter Inductor (L) | 1mH |
| Filter Capacitor (C) | 1.0132µF |
| Resistive Load | 150Ω |
| RL Load | 150Ω + 100mH |
| Power Switch | Infineon SPA11N60C3 CoolMOS (650V, 340mΩ) |

---

## Key Design Decisions

**MOSFET Selection — Infineon SPA11N60C3 CoolMOS**
Selected for its 650V Vds rating (3.25× derating over 200V bus to handle switching transients), 340mΩ Ron for low conduction losses, and CoolMOS superjunction structure optimised for high-voltage switching applications. Gate charge of 45nC is manageable at 50kHz.

**Dead Time — 300ns**
Inserted between complementary gate signals to prevent shoot-through — the condition where both MOSFETs in the same leg conduct simultaneously, creating a direct short across the DC bus. Dead time was implemented via a 0.06V offset in the SPWM comparator logic, corresponding to 300ns at a 0.2V/µs triangle carrier slope. Verified in simulation by observing the 300ns blanking interval in gate waveforms.

**LC Filter Cutoff at 5kHz**
Placed one decade below the 50kHz switching frequency to achieve strong harmonic attenuation (−40dB/decade, second order). This allows the 50Hz fundamental to pass with negligible attenuation while rejecting switching harmonics. Characteristic impedance Z₀ = √(L/C) ≈ 31.4Ω; load resistance of 150Ω was chosen to remain well above Z₀ to prevent filter response distortion.

---

## Simulation Results

| Simulation | Key Result | Status |
|---|---|---|
| Transient — AC output (resistive) | 157.4V peak, clean 50Hz sine | ✅ Pass |
| Dead time verification | 300ns blanking confirmed, body diode conduction visible | ✅ Pass |
| THD — resistive load | 2.45% (target <5%) | ✅ Pass |
| THD — RL load | 2.54% | ✅ Pass |
| Phase lag — RL load | Current lags voltage, visible in waveform | ✅ Pass |
| Efficiency (simulated) | 99.03% (realistic estimate: 95–97%) | ✅ Pass |

**THD Harmonic Breakdown (Resistive Load):**

| Harmonic | Frequency | Normalised Amplitude |
|---|---|---|
| 1st (fundamental) | 50Hz | 1.000 (157.4V) |
| 3rd | 150Hz | 0.870% |
| 5th | 250Hz | 1.653% |
| 7th | 350Hz | 0.264% |
| 9th | 450Hz | 0.718% |
| Even harmonics | — | ~0% (cancelled by symmetric topology) |

Even harmonics are suppressed by the symmetric H-Bridge topology. The 5th harmonic dominates — characteristic of dead-time distortion in SPWM inverters.

---

## Waveform Results

### AC Output — Resistive Load
![AC Output Resistive](Results/01_AC_Output_Resistive.png)
*Clean 50Hz sine wave, ±157.4V peak, centered at 0V. Measured as differential V(v_out)−V(phase_b).*

### Dead Time Verification
![Dead Time Verification](Results/02_Dead_Time_Verification.png)
*Zoomed to single switching transition. Both phase nodes simultaneously at 0V during 300ns dead time interval. Body diode conduction visible as intermediate voltage plateau.*

### SPICE Log — Resistive Load THD and Efficiency
![SPICE Log Resistive](Results/03_SPICE_Log_Resistive_THD.png)
*THD = 2.45%, DC offset = −0.002V (essentially zero), efficiency = 99.03%.*

### RL Load — Voltage and Current
![RL Load](Results/04_RL_Load_Voltage_Current.png)
*Voltage (red) and load current (green) under RL load. Current lags voltage due to inductive load. Load: 150Ω + 100mH.*

### SPICE Log — RL Load THD and Efficiency
![SPICE Log RL](Results/05_SPICE_Log_RL_THD.png)
*THD = 2.54% under RL load — increase of only 0.09% versus resistive load, confirming filter effectiveness across load types.*

---

## Output Voltage — Why 8% Below Theoretical

Theoretical peak: M × VDC = 0.85 × 200 = 170V. Measured: 157.4V. The 7.7% deficit comes from three sources:

- **Dead time voltage loss:** 300ns dead time per transition, two transitions per switching cycle, at 50kHz — effective duty cycle reduction of ~3%, corresponding to ~5.1V loss.
- **MOSFET conduction drop:** Two MOSFETs in series per half cycle, Ron = 340mΩ each, at ~1A load — ~0.68V total.
- **Inductor impedance at 50Hz:** XL = 2π × 50 × 0.001 = 0.314Ω, minor voltage division with load.

These are expected, predictable, and consistent with simulation results.

---

## Efficiency — Simulation vs Reality

Simulated efficiency of 99% is optimistic because:

- Behavioural voltage sources (B-sources) drive MOSFET gates with zero source impedance, making switching transitions unrealistically fast and eliminating switching loss overlap.
- Gate drive losses (Qg × Vgs × fsw × 4 MOSFETs = 45nC × 10V × 50kHz × 4 = 90mW) are not captured in the DC supply current measurement.
- Ideal inductors in LTSpice have no core loss. Real inductors at 50kHz switching frequency have measurable core losses.

Realistic efficiency estimate accounting for switching losses (~1.2W), gate drive (0.09W), and core losses: **95–97%.**

---

## How to Run

1. Install LTSpice XVII (free from Analog Devices)
2. Open `LTSpice/Draft5.asc`
3. Run simulation — `.tran 0 60ms 0 10ns` is already set
4. Probe `V(v_out,phase_b)` for differential AC output
5. View → SPICE Error Log for THD and efficiency measurements

---

## Tools Used

- LTSpice XVII 24.1.10
- Infineon SPA11N60C3 SPICE model (included in LTSpice standard library)
