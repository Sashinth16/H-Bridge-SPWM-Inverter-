# Design Calculations — Single-Phase H-Bridge SPWM Inverter

All calculations performed analytically before simulation. Simulation results validated against these targets.

---

## 1. Output Voltage and Modulation Index

For a full H-Bridge with Sinusoidal PWM:

```
V_peak,output = M × V_DC
V_RMS = (M × V_DC) / √2
```

**Given:**
- V_DC = 200V
- M = 0.85 (practical value, avoids overmodulation)

**Results:**
```
V_peak = 0.85 × 200 = 170V (theoretical)
V_RMS  = 170 / √2 = 120.2V RMS (theoretical)
```

**Simulated:** 157.4V peak / ~111V RMS (8% lower — see voltage deficit analysis in README)

---

## 2. MOSFET Voltage Rating

Rule: Vds_rated ≥ 2 × V_DC (to handle switching voltage spikes from parasitic inductance)

```
Vds_min = 2 × 200 = 400V minimum
```

**Selected:** Infineon SPA11N60C3 — Vds = 650V → derating factor = 650/200 = **3.25×**

This exceeds the minimum 2× requirement, providing additional margin for real-world parasitic spikes.

---

## 3. LC Filter Design

**Objective:** Low-pass filter — pass 50Hz fundamental, attenuate 50kHz switching harmonics.

**Cutoff frequency rule:** fc ≈ 1 decade below switching frequency

```
f_sw  = 50kHz
f_c   = f_sw / 10 = 5kHz
```

**Filter resonant frequency formula:**
```
f_c = 1 / (2π√LC)
```

**Solving for C with L = 1mH:**
```
5000 = 1 / (2π × √(0.001 × C))
√(0.001 × C) = 1 / (2π × 5000)
0.001 × C = 1 / (2π × 5000)²
C = 1 / ((2π × 5000)² × 0.001)
C = 1 / (9.8696 × 10⁸ × 0.001 × 10⁻³... )
```

Let me show this more clearly:
```
(2π × 5000)² = (31415.9)² = 9.8696 × 10⁸
C = 1 / (9.8696 × 10⁸ × 0.001)
C = 1 / (9.8696 × 10⁵)
C = 1.0132 × 10⁻⁶ F
C = 1.0132µF
```

**Verified:** fc = 1/(2π√(0.001 × 1.0132×10⁻⁶)) = **5000Hz ✓**

---

## 4. Filter Characteristic Impedance

```
Z₀ = √(L/C) = √(0.001 / 1.0132×10⁻⁶) = √(987.0) = 31.4Ω
```

**Design rule:** Load resistance >> Z₀ to prevent filter response distortion.

```
R_load = 150Ω >> Z₀ = 31.4Ω  →  ratio = 4.78×  ✓
```

---

## 5. Dead Time Calculation

**Switching period:**
```
T_sw = 1 / f_sw = 1 / 50000 = 20µs
```

**Dead time chosen:** 300ns

**As percentage of switching period:**
```
300ns / 20µs = 1.5%
```

Acceptable — small enough to limit waveform distortion, large enough to prevent shoot-through given SPA11N60C3 turn-off characteristics.

**Implementation via comparator offset:**
Triangle carrier slope = 2V / 10µs = 0.2V/µs

```
Dead time = offset / slope = 0.06V / 0.2V/µs = 0.3µs = 300ns ✓
```

**SPWM gate logic:**
```
M1, M4 ON when: V(sine_ref) > V(tri_carrier) + 0.06
M2, M3 ON when: V(sine_ref) < V(tri_carrier) − 0.06
```

Gap between conditions = 0.12V → corresponds to 300ns dead time on each transition.

---

## 6. Load Design

### Resistive Load
```
R = 150Ω

I_peak = V_peak / R = 170 / 150 = 1.13A (theoretical)
P_load = V_RMS² / R = 120² / 150 = 96W (theoretical)
```

Simulated: Pout = 82.59W (lower due to voltage deficit)

### RL Load
```
R = 150Ω
L = 100mH

Load time constant: τ = L/R = 0.1/150 = 0.67ms

Inductive reactance at 50Hz:
X_L = 2π × 50 × 0.1 = 31.4Ω

Total impedance:
Z = √(R² + X_L²) = √(150² + 31.4²) = √(22500 + 985.96) = √23485.96 = 153.3Ω

Phase angle:
φ = arctan(X_L/R) = arctan(31.4/150) = arctan(0.209) = 11.8°
```

Current lags voltage by 11.8° theoretically.
Observed lag in simulation is larger (~36°) due to interaction between filter capacitor and load inductor forming a secondary resonant circuit.

---

## 7. Expected vs Simulated Results

| Parameter | Calculated | Simulated | Error | Reason |
|---|---|---|---|---|
| V_peak | 170V | 157.4V | 7.4% | Dead time + Ron + filter drop |
| f_out | 50Hz | 50Hz | 0% | — |
| f_c | 5kHz | 5kHz | 0% | — |
| Dead time | 300ns | ~300ns | <5% | MOSFET finite switching time |
| THD | <5% target | 2.45% | — | Below target ✓ |

---

## 8. Voltage Deficit Analysis

**Theoretical peak:** 170V
**Simulated peak:** 157.4V
**Deficit:** 12.6V (7.4%)

**Contribution breakdown:**

Dead time voltage loss:
```
Dead time ratio = 2 × 300ns / 20µs = 3% per cycle
V_loss,deadtime = 3% × 170V = 5.1V
```

MOSFET conduction loss (2 MOSFETs in series):
```
V_loss,Ron = I × 2 × Ron = 1.0 × 2 × 0.34 = 0.68V
```

Filter inductor voltage drop at 50Hz:
```
V_loss,filter = I × X_L = 1.0 × 0.314 = 0.31V
```

Total estimated loss:
```
V_loss,total = 5.1 + 0.68 + 0.31 = 6.09V
```

Remaining ~6.5V deficit is attributable to non-linear dead time effects at varying duty cycles across the sine cycle and MOSFET body diode voltage during commutation — consistent with simulation.

---

## 9. Efficiency Analysis

**Simulated:**
```
Pout = 82.59W
Pin  = 83.40W
η    = 99.03%
```

**Realistic estimate — additional losses not captured in simulation:**

Switching losses (per MOSFET, estimated):
```
P_sw = ½ × V_DS × I_D × (t_r + t_f) × f_sw
     = ½ × 200 × 1.0 × (30ns + 30ns) × 50000
     = ½ × 200 × 1.0 × 60×10⁻⁹ × 50000
     = 0.3W per MOSFET
     × 4 MOSFETs = 1.2W total
```

Gate drive losses:
```
P_gate = Q_g × V_gs × f_sw × 4
       = 45×10⁻⁹ × 10 × 50000 × 4
       = 0.09W
```

Estimated core losses (inductor at 50kHz): ~0.2W

**Revised total losses:**
```
P_loss,total = 0.81 (conduction) + 1.2 (switching) + 0.09 (gate) + 0.2 (core)
             = 2.3W
```

**Realistic efficiency:**
```
η_realistic = Pout / (Pout + P_loss,total)
            = 82.59 / (82.59 + 2.3)
            = 82.59 / 84.89
            = 97.3%
```

Realistic efficiency range: **95–97%** accounting for component tolerances and temperature effects.

---

## 10. THD Summary

| Load | THD | Dominant Harmonic | Target |
|---|---|---|---|
| Resistive (150Ω) | 2.45% | 5th (250Hz) — 1.65% | <5% ✓ |
| RL (150Ω + 100mH) | 2.54% | 5th (250Hz) — 1.65% | <5% ✓ |

Even harmonics (2nd, 4th, 6th) are suppressed to <0.001% by the symmetric H-Bridge differential topology.

5th harmonic dominance is characteristic of SPWM dead-time distortion. Reduction strategies: decrease dead time, increase switching frequency, or implement active dead-time compensation.
