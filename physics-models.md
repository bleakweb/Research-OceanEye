# physics-models.md — inputs, models, and the decision chain

> **The framing, stated once.** The physics does *not* hand you a waveform. It hands you **constraints and a cost**. You still have to declare what you are optimising for, and that declaration is an engineering choice — it is your team's actual contribution, and it is the thing to defend at the judging table.
>
> What is genuinely settled: given (temperature, salinity, depth, turbidity, required range), the physics tells you **exactly how much signal you lose** at any frequency, and **exactly how fine an image** any bandwidth can produce. Those are equations with published coefficients and known validity ranges. No water needed.
>
> What is your choice: the **objective function**. This document proposes *"maximise resolution subject to meeting a detection-SNR threshold, then minimise energy"* — and that single sentence is what your adaptation logic implements.
>
> What is genuinely hard, and genuinely yours: doing the synthesis **in real time, in hardware, on a battery budget**. The physics is the easy half. R-0.10, R-0.12 and R-1.6 are the hard half.

---

## 0. Two corrections to earlier statements in this project

Research turned up two things I had wrong. Both are now fixed in `glossary.md`; recording them here so nobody works from the old version.

**Correction 1 — what turbidity actually does to your signal.** I said mud attenuates via Rayleigh scattering rising as f⁴. That is **wrong for attenuation**. For fine mud (1–60 µm) at 100–500 kHz, **viscous absorption dominates by four to ten orders of magnitude**. Scattering only takes over above roughly **210 µm at 500 kHz** and **850 µm at 100 kHz** — coarse sand, not mud. And the viscous term goes as roughly **f^0.5–0.7** in our band, not f². Rayleigh f⁴ *is* correct for **backscatter** (the echo strength), which is a different quantity — don't let anyone conflate them, including me.

**Correction 2 — fractional bandwidth.** I assumed transducers give 30–50%. Reality: good broadband side-scan transducers support **20–35%** (Neptune Sonar: 34.8% at 115 kHz, 20% at 500 kHz), and **real shipping systems operate at about 8%** — EdgeTech and Klein both back-compute to ~8% from their published range resolutions. They spend the available bandwidth on SNR, not resolution. **Design to 25% capability, 10% operating.** If your team computes a resolution five times better than a Klein 4000, a reviewer will spot it immediately.

---

## 1. Input parameters

### 1.1 What the PS names, and what the physics actually needs

The PS names four (B-7): **depth, turbidity, temperature, salinity**. Those four are necessary but not sufficient — the models also need a **required range**, because "is this frequency good enough?" is meaningless without saying *good enough at what distance*.

| # | Parameter | Symbol | Unit | Realistic range | Named in PS? | What it physically drives |
|---|---|---|---|---|---|---|
| 1 | Water temperature | T | °C | 0–35 | ✅ B-7 | Sound speed; both relaxation frequencies; absorption |
| 2 | Salinity | S | ppt (PSU) | 5–38 | ✅ B-7 | Sound speed; **MgSO₄ absorption term (dominant in our band)** |
| 3 | Depth | z | m | 0–200 | ✅ B-7 | Sound speed (weakly); absorption (negligibly below 50 m) |
| 4 | Turbidity | — | NTU | 0.1–2000+ | ✅ B-7 | Excess viscous attenuation from suspended sediment |
| 5 | **Altitude above seabed** | h | m | 5–50 | ❌ **implied** | Sets the required slant range — *the actual driver of the frequency decision* |
| 6 | pH | — | — | 7.7–8.3 | ❌ | Boric-acid absorption term — **negligible above 10 kHz, drop it** |

**On parameter 5.** This is the gap in the PS's list and worth raising in your presentation as a piece of independent thinking. Depth (how deep the *vehicle* is) barely affects anything in our band. **Altitude** (how far above the *seabed*) sets how far the ping must travel, and that is what actually decides the frequency. Standard side-scan practice: fly at **10–20% of the range scale** — so a 100 m swath means 10–20 m altitude. Either derive range from a depth-plus-terrain assumption, or add altitude as a fifth input and say why.

**On parameter 3.** Depth enters absorption as `exp(−z/6)` with z in **kilometres**. At 5 m vs 10 m that's 0.99917 vs 0.99833 — a 0.08% difference. **Below ~50 m, set the depth terms to 1.0 and lose nothing.** Keep depth as an input because the PS names it and because it feeds sound speed and range geometry, but be honest that its effect on absorption is nil. Judges respect that more than fake sensitivity.

**On parameter 2.** Salinity is the sleeper. Dropping S from 35 → 15 ppt (seawater → brackish estuary) **halves** the absorption at 100 kHz. Over a 100 m two-way path that's a **15 dB error at 500 kHz** if you hard-code it. Do not hard-code salinity.

### 1.2 Sensors (bench-practical, India-available)

Constraint: R-2.5 requires the MCU to read via **ADC**, so analog outputs are preferred.

| Parameter | Recommended part | Output | ₹ | Notes |
|---|---|---|---|---|
| Temperature | **NTC 10K B3950 stainless probe** | Analog (resistance → divider) | ~55 | Genuinely waterproof, you set the swing with the series resistor. Add a **DS18B20** (~₹90, 1-Wire) as a calibration cross-check — having both an analog and a digital path in one subsystem demos well |
| Salinity | **Potentiometer**, labelled 5–40 ppt | Analog | ~30 | ⚠️ **Corrected — see below.** No affordable probe covers the seawater range |
| Turbidity | **Generic analog turbidity module** (Robocraze) | Analog 0–4.5 V | ~555 | Buy two — the LED/phototransistor pair is the most failure-prone item on the list. Add milk drop by drop for the most visually convincing demo you have |
| Depth | **Potentiometer**, or MPX5010DP if budget allows | Analog | 0 / ~1,959 | 0–10 kPa ≈ 0–1 m water column, which is exactly bench scale. But a water column on a demo table is the likeliest thing to leak. This is the one channel where a pot costs nothing in credibility |

> **⚠️ Correction to an earlier recommendation in this project.** I previously proposed the DFRobot SEN0244 TDS module for salinity. **It cannot do the job.** Its output saturates at 2.3 V, which through DFRobot's own cubic is EC = 2.24 mS/cm = **1.15 ppt**. Seawater at 35 ppt is 53.06 mS/cm — the part is short by a factor of about 30, and the DFR0300 K=1 probe tops out near 20 mS/cm so it does not rescue this either (seawater needs a K=10 cell). Applying YAGNI: **salinity becomes a potentiometer channel**, which the PS explicitly permits (R-0.5, R-2.3). Keep the EC→salinity conversion chain in the firmware behind a compile flag so the code is probe-ready; the verified formulas are in `signal-chain.md` §2.2.

**Two things that will win marks:**

1. **Jumper a 10K pot in parallel on every ADC channel** (~₹120 for four pots and headers). If a sensor dies in transit or venue lighting wrecks the turbidity reading, flip a jumper and the demo continues. It also doubles as a legitimate range-sweep test rig. The PS explicitly permits potentiometers (R-0.5, R-2.3), so this is sanctioned, not a hack.
2. **Put the 5 V→3.3 V dividers on the board, not on a breadboard**, and document one scaling table: sensor → raw V → divided V → ADC counts → engineering units. That table is the cheapest possible evidence you understood the "read via ADC" constraint rather than just plugging in modules.

**Do not buy:** DFRobot EC PRO / DFR0300 (₹7,668+, needs 1413 µS/cm and 12.88 mS/cm calibration standards you can't source in hackathon week); any 4–20 mA submersible transmitter (₹14,500, 24 V loop supply); MPX5700DP (0–700 kPa, over-ranged 500× — 1 m of water is 1.4% of full scale). Name all three on a "path to production" slide as the correct field parts, and say why they're wrong for a bench.

---

## 2. The model chain

Five models, in order. Each is a published equation with a validity range. Numbers here are computed, not quoted from memory.

### Model 1 — Sound speed: Mackenzie (1981)

```
c = 1448.96 + 4.591·T − 5.304e-2·T² + 2.374e-4·T³
    + 1.340·(S − 35) + 1.630e-2·D + 1.675e-7·D²
    − 1.025e-2·T·(S − 35) − 7.139e-13·T·D³
```
`c` in m/s · `T` in °C · `S` in ppt · **`D` in metres** · fit standard error 0.070 m/s
**Validity:** T −2…30 °C, **S 25…40 ppt**, D 0…8000 m.

⚠️ Note the salinity floor. For brackish estuary work (S = 10–15) this is an extrapolation; the `1.340·(S−35)` term extrapolates gracefully but say so. **Coppens (1981)** covers S 0–45 if you want to be rigorous.

**Sensitivity:** ≈3.2 m/s per °C · ≈1.19 m/s per ppt · ≈1.6 m/s per 100 m depth. Temperature dominates.

**Does c = 1500 matter?** Two different answers, and the distinction is worth making explicitly:

- **For resolution** (`Δr = c/2B`): across a realistic coastal envelope c spans 1438–1552 m/s, a 4% error. On a 15 mm resolution cell that's half a millimetre. **Irrelevant.**
- **For absolute ranging** (`R = c·t/2`): the same 4% is **4 metres of position error at 100 m range**, and it skews slant-to-ground-range correction. **Not irrelevant.**

Since you already carry T and S at runtime for Model 2, feeding them into Mackenzie costs nine multiply-adds. Do it.

### Model 2 — Seawater absorption: Ainslie & McColm (1998)

```
α = 0.106 · (f1·f²)/(f² + f1²) · exp((pH − 8)/0.56)              [boric acid]
  + 0.52 · (1 + T/43) · (S/35) · (f2·f²)/(f² + f2²) · exp(−z/6)  [magnesium sulphate]
  + 0.00049 · f² · exp(−(T/27 + z/17))                           [pure water viscosity]

f1 = 0.78 · √(S/35) · exp(T/26)   [kHz]
f2 = 42 · exp(T/17)               [kHz]
```
`α` in **dB/km** · `f, f1, f2` in **kHz** · `T` in °C · `S` in ppt · **`z` in KILOMETRES** (10 m → z = 0.01)
**Validity:** 100 Hz–1 MHz, T −6…35 °C, S **5…50 ppt**, D 0…7 km, pH 7.7…8.3.

**Use this one, not Francois–Garrison.** F&G is the more accurate reference model in general, but in the 10–500 kHz band it is only validated for **S 30–35 ppt and T < 22 °C** — which excludes both your brackish estuary and warm Indian coastal water. A&M covers your whole envelope. (The two agree to ~2% at 100 kHz and ~6% at 500 kHz where both are valid, which is good mutual validation since they share no coefficients.)

**Firmware note:** the boric acid term contributes <0.6% in our band. **Drop term 1 and the pH input entirely.** Below 50 m depth, set both exponentials to 1.0. What remains is two multiplies and a divide.

**Computed values** (pH 8):

| Condition | 100 kHz | 500 kHz | Breakdown at 500 kHz (boric / MgSO₄ / water) |
|---|---|---|---|
| S=35, T=10 °C, z=10 m | **34.3** | **132.0** | 0.12 / 47.3 / 84.5 |
| S=35, T=25 °C, z=10 m | **36.7** | **181.1** | 0.22 / 132.4 / 48.5 |
| S=15, T=25 °C, z=5 m (brackish) | **16.9** | **105.4** | 0.14 / 56.8 / 48.5 |

dB/km. Note how the dominant term *swaps* between cold and warm water at 500 kHz — pure-water viscosity dominates cold, MgSO₄ dominates warm. That is a real, defensible piece of physics to point at.

### Model 3 — Excess attenuation from suspended sediment

Two additive terms, both **linear in mass concentration** in the dilute limit:

```
α_sed = (ζ_sv + ζ_ss) · M_s          [Np/m];   × 8686 → dB/km
M_s = suspended sediment mass concentration [kg/m³]  (1 kg/m³ = 1000 mg/L)
```

**Viscous absorption** (Urick 1948 form) — **the dominant term for mud**:
```
γ = √(πF/ν)                          ν ≈ 1.0e-6 m²/s
s = (9/(2γd))·(1 + 2/(γd))           d = particle diameter
T_u = 1/2 + 9/(2γd)
ζ_sv = (k/(2ρ_s))·(σ−1)²·[ s / (s² + (σ + T_u)²) ]
       k = 2πF/c,  σ = ρ_s/ρ_w,  ρ_s ≈ 2650 kg/m³
```

**Scattering** (negligible for mud, included for completeness):
```
x = ka,   χ = 0.29·x⁴ / (0.95 + 1.28·x² + 0.25·x⁴)
ζ_ss = 3χ / (4·ρ_s·a)                a = particle radius
```

**Computed excess attenuation at 100 mg/L**, dB/km:

| Particle d | 100 kHz | 500 kHz |
|---|---|---|
| 1 µm | 4.3 | 61 |
| 2 µm | 10.9 | **81 ← peak** |
| 5 µm | **16.0 ← peak** | 53 |
| 10 µm | 11.5 | 29.7 |
| 20 µm | 6.6 | 15.6 |
| 60 µm | 2.3 | 5.4 |

Note the resonance: viscous loss peaks near d ≈ 5 µm at 100 kHz and d ≈ 2 µm at 500 kHz. **Estuarine mud sits right on the peak** — worst case.

**How much does turbidity actually matter?** Taking d = 15 µm as representative, against seawater absorption:

| SSC | Sediment @100 kHz | vs seawater | Sediment @500 kHz | vs seawater |
|---|---|---|---|---|
| 10 mg/L | 0.8 | 2% | 2.0 | 2% |
| 100 mg/L | 8.4 | **24%** | 20.5 | **15%** |
| 500 mg/L | 42 | 122% | 102 | 77% |
| 1000 mg/L | 84 | 245% | 205 | 155% |

**Two counterintuitive findings worth putting on a slide:**

1. At ordinary turbidity (≤100 mg/L) sediment is a **15–25% correction, not a dominant effect**. It only takes over above ~500 mg/L.
2. Turbidity is **relatively more punishing at 100 kHz than at 500 kHz** — because seawater absorption climbs as ~f^1.5 while sediment loss climbs as only ~f^0.6. This cuts against the naive "go low-frequency in mud" intuition. Low frequency still wins on the absolute budget, but *not* for the reason most people assume. Being able to say that is a differentiator.

**Turbidity units.** There is no universal NTU→mg/L conversion; the relationship is instrument- and site-specific (Alberta Environment requires ≥20 paired samples and R² ≥ 0.85 before allowing one). Usable engineering figure: **SSC ≈ 1–2 mg/L per NTU** for fine mineral silt, up to 4 for coarser or organic-rich material. **State this as an assumption with a citation, don't hide it.**

**Typical environments:**

| Environment | NTU | SSC |
|---|---|---|
| Clear shallow coral reef | 0.1–5 | ~5 mg/L |
| Reef during resuspension / cyclone | 20–100+ | 20–200 mg/L |
| Coastal water generally | 1–20 | 1–50 mg/L |
| Muddy tidal estuary (background) | 10–100 | <100 mg/L |
| **Estuarine turbidity maximum** | hundreds–thousands | **up to several g/L** |

That last row is a factor of a thousand from the reef case. **That span is your adaptive transmitter's whole reason to exist** — quote it.

**Honesty note for the report:** real estuarine mud **flocculates**, and flocs have far lower effective density contrast (σ→1), which *reduces* viscous attenuation well below these rigid-sphere numbers. Treat the table as a conservative upper bound and say so.

### Model 4 — Transmission loss

```
TL = 20·log₁₀(R) + (α_seawater + α_sed)·R/1000        [dB, single-way]
```
`R` in metres, `α` in dB/km. Spherical spreading + absorption. **Convention: TL is single-way; the active sonar equation uses 2·TL** for the monostatic round trip.

### Model 5 — The sonar equation

**Noise-limited** (detecting a discrete target — wreck, mine, pipeline):
```
SNR = SL − 2·TL + TS − (NL − DI) + G_p      ≥ DT
```

**Reverberation-limited** (ordinary seafloor imaging — the normal side-scan case):
```
RL = SL − 2·TL + Sb + 10·log₁₀(A)
A = R · φ_az · (c·τ_eff / 2) / cos(θ_g)
SNR_rev = TS − Sb − 10·log₁₀(A)
```

⚠️ **Look at that last line.** In the reverberation-limited case **SL and TL cancel out**. Shouting louder does *not* improve a reverberation-limited image. The only levers are a smaller footprint (narrower beam, shorter effective pulse) and grazing-angle geometry.

**This is a genuinely important result for your amplitude logic (R-2.9).** It means amplitude should be set to the *minimum* that clears the noise floor with margin — pushing it higher burns battery and buys nothing once you are reverberation-limited. That is a defensible, physics-derived rule, and it's a much better answer than "muddy water so turn it up."

**Component terms:**

- **Noise level (thermal-dominated above ~100 kHz):** `NL₀ = −15 + 20·log₁₀(f_kHz)` dB re 1 µPa²/Hz, then add `10·log₁₀(B)`. Verified against the fluctuation-dissipation result to within 0.36 dB.

  | f | Spectrum level | B = 10 kHz | B = 50 kHz |
  |---|---|---|---|
  | 100 kHz | 25.0 | 65.0 | 72.0 |
  | 500 kHz | 39.0 | 79.0 | 86.0 |

  Caveat: this is a **floor**, not a prediction. Snapping shrimp extend well past 100 kHz in warm coastal water and can add 10–20 dB. Budget margin.

- **Directivity index:** `DI = 10·log₁₀(4π / (φ_az·φ_el))`, beamwidths in radians.
- **Processing gain:** `G_p = 10·log₁₀(B·τ)` — the time-bandwidth product. **This is the highest-value term in the whole budget**, because it buys SNR *and* resolution simultaneously, whereas a longer CW pulse trades one against the other.
- **Bottom backscattering strength (Lambert):** `Sb ≈ 10·log₁₀(µ) + 10·log₁₀(sin²θ_g)`, with µ ≈ −27 dB for sand, ≈ −35 dB for mud.

### Model 6 — Resolution

```
Range resolution (compressed):   Δr = c / (2B)
Range resolution (plain pulse):  Δr = c·τ / 2
Across-track on the seabed:      R_y = c·τ_eff / (2·cos β)      β = grazing angle
Along-track:                     R_x = R · φ_az                  φ_az in radians
Bandwidth constraint:            B ≤ β_frac · fc
```

The `cos β` term in across-track resolution is dropped by most student treatments — keeping it is a cheap credibility signal.

**With realistic fractional bandwidth** (10% operating, per the correction in §0):

| fc | B at 10% | Δr | Real-world check |
|---|---|---|---|
| 100 kHz | 10 kHz | 7.5 cm | Klein 4000 @ 100 kHz quotes **9.6 cm** ✓ |
| 200 kHz | 20 kHz | 3.75 cm | |
| 400 kHz | 40 kHz | 1.9 cm | EdgeTech 4125i @ 400 kHz quotes **2.3 cm** ✓ |
| 500 kHz | 50 kHz | 1.5 cm | |

Our computed values land within 25% of shipping commercial hardware. **That agreement is worth showing** — it's independent evidence the model chain is sane, and it costs one slide.

**This is the rigorous version of B-9's "blurry image".** Low centre frequency doesn't blur the image directly — it *caps the available bandwidth*, and bandwidth is what sets resolution. Three equations, no hand-waving.

---

## 3. The decision algorithm

Everything above is published physics. **This section is the part you invent**, and it is what R-2.6 asks for.

**Objective:** *maximise image resolution, subject to achieving detection SNR at the required range with margin, then minimise transmitted energy.*

```
INPUTS:  T, S, z, turbidity_NTU, altitude h
DERIVED: R_max = required slant range   (≈ 5–10 × h, standard side-scan practice)
         c     = Mackenzie(T, S, z)
         M_s   = k_NTU · turbidity_NTU           (k_NTU ≈ 1–2 mg/L per NTU)

STEP 1 — pick centre frequency (the range/resolution trade)
  for fc in [500, 400, 300, 200, 150, 100] kHz:      # highest first
      α_total = AinslieMcColm(fc, T, S, z) + sediment(fc, M_s, d̄)
      TL      = 20·log10(R_max) + α_total·R_max/1000
      B_trial = 0.10 · fc
      G_p     = 10·log10(B_trial · τ_nominal)
      SNR     = SL_max − 2·TL + TS − NL(fc, B_trial) + DI + G_p
      if SNR ≥ DT + margin:  choose this fc and STOP
  # if none qualify, take the lowest fc and flag reduced range

STEP 2 — bandwidth  (R-2.7)
  B = β_frac · fc            with β_frac = 0.10 operating, 0.25 transducer ceiling
  → resolution Δr = c / (2B)

STEP 3 — pulse duration  (R-2.8)
  τ = required BT / B         where BT comes from the G_p needed in step 1
  bounded by:  τ ≥ 1/B                        (can't be shorter than the inverse bandwidth)
               τ ≤ 2·R_min/c                  (blind range — don't still be transmitting
                                               when the nearest echo returns)
               τ · PRF ≤ duty_max             (battery, R-0.8)

STEP 4 — amplitude  (R-2.9)
  A = minimum level meeting DT + margin at R_max
  # NOT maximum. See Model 5: once reverberation-limited, extra SL cancels out
  # and buys nothing but battery drain.
```

**Why this algorithm is defensible:** every branch traces to an equation above, the objective is stated in one sentence, and step 4 encodes a non-obvious physical result. It is not a lookup table with plausible-looking numbers — and a judge can tell the difference in about thirty seconds.

**Keep a lookup table anyway**, as a fallback path if the solver misbehaves on stage. Compute it *from* the algorithm offline so the two always agree.

---

## 4. The two named scenarios, end to end

These are the demo cases (R-2.4). Numbers below are illustrative of the chain, not final — regenerate them once you fix SL, DI, TS and DT for your own link budget.

### "Entering Clear Shallow Reef"

| | |
|---|---|
| **Sensed** | T = 28 °C, S = 35 ppt, z = 8 m, turbidity = 2 NTU (≈3 mg/L), altitude = 10 m |
| Sound speed | c ≈ 1543 m/s |
| Seawater absorption @ 500 kHz | ≈ 190 dB/km |
| Sediment attenuation | ≈ 0.6 dB/km — **negligible** |
| Required range | ≈ 75 m |
| **Decision** | fc = **500 kHz**, B = **50 kHz**, τ = **short**, amplitude = **low** |
| Resolution | Δr ≈ **1.5 cm** |
| **Why** | Clear water, short range. Nothing is stopping the high frequency, so take the resolution. Range demand is modest, so drop amplitude and save battery. |

### "Entering Muddy Estuary"

| | |
|---|---|
| **Sensed** | T = 26 °C, S = 15 ppt (brackish), z = 6 m, turbidity = 400 NTU (≈600 mg/L), altitude = 15 m |
| Sound speed | c ≈ 1516 m/s |
| Seawater absorption | ≈ 17 dB/km @ 100 kHz, ≈ 107 dB/km @ 500 kHz (low salinity helps a lot) |
| Sediment attenuation (d ≈ 15 µm, 600 mg/L) | ≈ **50 dB/km @ 100 kHz**, ≈ **123 dB/km @ 500 kHz** — now dominant |
| Total @ 500 kHz | ≈ 230 dB/km → **34 dB lost over just 150 m one-way**. Unusable. |
| Total @ 100 kHz | ≈ 67 dB/km → **10 dB over 150 m**. Workable. |
| Required range | ≈ 110 m |
| **Decision** | fc = **100 kHz**, B = **10 kHz**, τ = **long** (BT gain to claw back SNR), amplitude = **high** |
| Resolution | Δr ≈ **7.6 cm** — five times worse, and that is the price of seeing anything at all |
| **Why** | The 500 kHz budget fails outright. Drop to 100 kHz, accept coarse resolution, buy SNR back with time-bandwidth product rather than raw power where possible. |

**The story those two tables tell** is exactly B-8 vs B-9 — but computed, with named models and traceable coefficients, instead of asserted. That is the difference between a project that repeats the problem statement and one that answers it.

---

## 5. Where this is weakest — say it before a judge finds it

| Weak link | Why | How to handle |
|---|---|---|
| **NTU → mg/L conversion** | Genuinely site-specific, no universal factor | State the assumed factor, cite the range (1–4), show sensitivity to it |
| **Particle size d̄** | You cannot measure it with a turbidity sensor, and viscous loss is strongly peaked in d | Assume 15 µm, show the d-sensitivity table from Model 3, call it a stated assumption |
| **Flocculation** | Real mud flocculates; flocs attenuate far less than rigid spheres | Say your numbers are a conservative upper bound, cite it |
| **SL, DI, TS, DT** | You have no transducer, so these are assumed | Pick published values from a real system, cite them, show the budget is *parametric* in them |
| **Mackenzie below S = 25** | Your estuary case is outside its validity | Acknowledge it; note Coppens covers S 0–45 if rigour is needed |
| **Snapping shrimp** | Can add 10–20 dB of noise past 100 kHz in warm coastal water | Carry explicit margin, mention it — this detail signals real reading |

Every one of these is a place where saying *"here is the assumption and here is its sensitivity"* scores higher than a confident number would.

---

## 6. What the firmware actually computes, and when

This is the architectural point that ties the physics back to R-0.10 and R-1.6. **There are two timescales, and they must not be confused.**

| Layer | Rate | What runs | Where |
|---|---|---|---|
| **Sensing + decision** | ~1–10 Hz | ADC reads, Mackenzie, Ainslie–McColm, sediment term, the §3 algorithm. Outputs three numbers: fc/B, τ, amplitude | Main loop. Floating point is fine here — you have millions of cycles |
| **Waveform synthesis** | 1–10 MSPS | Phase accumulator, sine LUT, window multiply, DMA to DAC | Timer + DMA. **CPU must not be in this path** |

**The whole physics engine runs at a few hertz.** Ainslie–McColm is about a dozen floating-point operations; even at 10 Hz on a 100 MHz MCU it is invisible in the power budget. **Do not let anyone talk you into putting it in the sample loop.**

The three outputs land in a parameter struct. The DMA waveform generator picks up new parameters at the next buffer boundary — which is exactly what makes R-2.6's "instantly" both true and measurable, and it's how you get "on the fly" (R-1.7) without a reset.

**Say this at the judging table.** The separation of a slow, floating-point decision layer from a fast, integer, DMA-driven synthesis layer *is* your architecture, and it is the direct answer to "how do you compute complex trigonometric values under strict real-time constraints without draining the battery" (R-0.12, R-0.11).

---

## 7. Sources

- Ainslie & McColm (1998), *JASA* 103(3):1671 — [abstract](https://pubs.aip.org/asa/jasa/article-abstract/103/3/1671/557895/A-simplified-formula-for-viscous-and-chemical)
- NPL, [Calculation of absorption of sound in seawater](https://resource.npl.co.uk/acoustics/techguides/seaabsorption/) — validity ranges
- NPL, [Speed of sound in sea water](https://resource.npl.co.uk/acoustics/techguides/soundseawater/underlying-phys.html) — Mackenzie coefficients
- Mackenzie (1981), *JASA* 70(3):807 — [PDF](https://pubs.aip.org/asa/jasa/article-pdf/70/3/807/11874746/807_1_online.pdf)
- Moore et al. (2016), *Water* 8(1):13, [Acoustic Properties of Suspended Sediment in Large Rivers](https://www.mdpi.com/2073-4441/8/1/13) — full Urick equations
- [On the concentration dependence of sound attenuation](https://pubs.aip.org/asa/jel/article/2/3/036002/2845717/On-the-concentration-dependence-of-sound), *JASA-EL* 2(3), 2022
- [Characterizing Flocculated Mineral Sediments with Acoustic Backscatter](https://pmc.ncbi.nlm.nih.gov/articles/PMC10603782/)
- [Fondriest — Turbidity, TSS and Water Clarity](https://www.fondriest.com/environmental-measurements/measurements/measuring-water-quality/turbidity-sensors-meters-and-methods/)
- [Alberta Environment — Conversion of NTU into TSS](https://www.alberta.ca/system/files/custom_downloaded_images/tr-conversion-of-nephelometric-turbidity-units.pdf)
- [Coastal Wiki — Estuarine turbidity maximum](https://www.coastalwiki.org/wiki/Estuarine_turbidity_maximum)
- MIT OCW 2.011, [Introduction to Sonar](https://ocw.mit.edu/courses/2-011-introduction-to-ocean-science-and-engineering-spring-2006/073d1246f6aac0102c6b29e0bfdfa5bc_hw5_sonar_leonar.pdf) — thermal noise, sonar equation
- [DOSITS — Active sonar equation](https://dosits.org/science/advanced-topics/sonar-equation/sonar-equation-example-active-sonar/)
- [EdgeTech 4125i](https://www.edgetech.com/wp-content/uploads/2023/04/4125i-Brochure-10-22-25.pdf) · [EdgeTech 4205](https://www.edgetech.com/wp-content/uploads/2023/04/0021769_Rev_J.pdf) · [Klein 4000](https://okeanus.com/wp-content/uploads/2025/08/klein_system_4000_rev0819_compressed.pdf) · [Klein 5900](https://www.klein.com/files/datasheets/S5900_2024.pdf) · [Kongsberg HISAS 1030](https://www.kongsberg.com/globalassets/kongsberg-discovery/naval/hisas/high-resolution-interferometric-synthetic-aperture-sonar---hisas-1030/)
- [Neptune Sonar 2020 transducer catalogue](https://www.neptune-sonar.co.uk/pdf/2020_catalogue.pdf) — real fractional bandwidths
- [MDPI *Remote Sensing* 15(23):5599](https://www.mdpi.com/2072-4292/15/23/5599) — side-scan resolution formulas, altitude practice
