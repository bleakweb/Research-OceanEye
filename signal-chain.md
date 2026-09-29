# signal-chain.md — sensors to waveform, end to end

Complete path from a voltage on an ADC pin to a sample leaving the DAC. Every stage, every parameter, every equation, with the code.

**Platform: RP2040 + external parallel DAC, via PIO and DMA.** Stages 1–9 and 11–13 are platform-independent; Stage 10 is RP2040-specific. See `decisions/d1-platform-rp2040.md`.

Companions: `objective.md` (requirement IDs), `physics-models.md` (the ocean models in detail), `glossary.md` (terms).

---

## 0. The chain

```
  SENSORS          ADC          UNITS        PHYSICS       DECISION      WAVE PARAMS
 ___________    _________    __________    __________    __________    ____________
| NTC       |  |         |  | T  degC  |  | c        |  |          |  | mod type   |
| EC probe  |->| 12-bit  |->| S  ppt   |->| alpha    |->| choose   |->| f0,f1,B,fc |
| turbidity |  | oversmp |  | NTU      |  | alpha_s  |  | fc, B    |  | tau, A     |
| pressure  |  | median  |  | z  m     |  | TL, NL   |  | tau, A   |  | window     |
|___________|  |_________|  |__________|  |__SNR_____|  |__________|  |____________|
                                                                            |
        ~10 Hz  ..........................................................  |
        ------------------------------------------------------------------- |
        1-10 MHz                                                            v
                                                                    ______________
   OUT <- BNC <- amp <- recon LPF <- I-V <- DAC <- GPIO <- PIO <- DMA <-| synth |
                                                          ^                |_______|
                                                   PIO clock divider
```

Two clock domains. **Everything above the dashed line runs at about 10 Hz in the main loop. Everything below runs at 1-10 MSPS in hardware.** They meet at exactly one place: a parameter struct handed over at a buffer boundary.

That separation *is* the architecture, and it is the direct answer to R-0.10, R-0.12, R-1.6 and R-2.6. Say it in one sentence at the judging table: *"the physics runs at 10 Hz in floating point, the synthesis runs at 5 MHz in integers under DMA, and the CPU is asleep in between."*

---

## Stage 1 — ADC acquisition

Four channels, R-2.5 requires ADC. Nothing here is time-critical; the whole point is to be slow and clean.

| Setting | Value | Why |
|---|---|---|
| Resolution | 12-bit | 4096 counts is ample for all four |
| Sample rate | 100 Hz per channel | 10x the decision rate, gives headroom for filtering |
| Oversampling | 64x accumulate, >>3 | Gains ~3 bits, kills ADC noise. STM32 does this in hardware |
| Trigger | Timer, DMA to buffer | Same DMA discipline as the DAC side. Free marks for consistency |
| Reference | External Vref, not VDD | Ratiometric sensors depend on it. See the trap below |

```c
volatile uint16_t adc_raw[4];      // DMA target, circular
// ADC1 -> DMA -> adc_raw[], triggered by TIM6 at 100 Hz. CPU never polls.
```

**Divider trap.** Three of the four sensors are 5 V parts feeding a 3.3 V ADC, so they need dividers. A resistive divider preserves ratiometricity **only if the divider's top rail and the ADC reference are the same node**. They are not. So either measure the 5 V rail on a fifth ADC channel and correct, or feed the sensors from a precision 5.0 V reference. A 1% wander on the 5 V rail is a 1% full-scale error on every sensor at once.

Keep total divider impedance under 10 kOhm so the sample-and-hold settles.

---

## Stage 2 — Counts to engineering units

All formulas below verified against datasheets and vendor source. Coefficients are literal.

### 2.1 Temperature — NTC 10K B3950

Topology: VCC -> Rs -> node -> NTC -> GND (NTC low side). **Use Rs = 13 kOhm, 0.1% metal film.** Optimal for 0-40 degC; a 1% resistor costs ~0.2 degC of offset on its own.

```c
#define ADC_N     4096.0f      // 2^12, NOT 4095
#define R_SERIES  13000.0f
#define SH_A      1.3097370563e-03f
#define SH_B      2.0364695280e-04f
#define SH_C      2.1562246122e-07f

float ntc_temperature_c(uint16_t adc)
{
    float r = R_SERIES * ((float)adc / (ADC_N - (float)adc));
    float l = logf(r);
    return 1.0f / (SH_A + SH_B*l + SH_C*l*l*l) - 273.15f;
}
```

**Use Steinhart-Hart, not the Beta equation.** Both cost one `logf`; Beta with the nameplate B=3950 is wrong by **+1.07 degC at 0 degC** because 3950 is the B25/50 figure, not the effective B across 0-40 degC (which is 3819). S-H with the coefficients above is within **0.017 degC** across the range. There is no reason to use Beta here.

If the NTC is wired high-side instead, the resistance formula inverts to `Rs * ((ADC_N - adc)/adc)`. Getting this backwards produces a monotonic, plausible, mirrored curve that fails silently.

### 2.2 Salinity — read this before buying anything

**Correction to earlier advice in this project.** I previously recommended the DFRobot SEN0244 TDS module for salinity. **It cannot do the job.** Its output saturates at 2.3 V, which through DFRobot's own cubic gives EC = 2.24 mS/cm = **1.15 ppt**. Seawater at 35 ppt is 53.06 mS/cm. The part is short by a factor of about 30. No maths fixes it, and the DFR0300 K=1 probe tops out around 20 mS/cm, so that does not fix it either — seawater needs a K=10 cell.

**Decision, applying YAGNI: use a potentiometer for salinity, labelled 5-40 ppt.** The PS explicitly permits potentiometers (R-0.5, R-2.3). No affordable probe covers the seawater range, and salinity is the input where a pot costs least credibility because judges know you cannot put 35 ppt seawater on a demo table without a mess.

```c
// Pot on 3.3V, full swing. Linear map to the physically meaningful range.
float pot_salinity_ppt(uint16_t adc)
{
    return 5.0f + (40.0f - 5.0f) * ((float)adc / ADC_N);
}
```

Keep the TDS conversion chain in the firmware behind a compile flag, so the code is honest about being probe-ready. If you do fit a real EC cell later:

```c
// DFRobot official cubic, GravityTDS.cpp. V in volts (5V domain). Result uS/cm.
float ec = (133.42f*V*V*V - 255.86f*V*V + 857.39f*V) * kValue;
float ec25 = ec / (1.0f + 0.02f*(temp_c - 25.0f));       // 0.02 verified; 0.0214 better for saline

// EC25 in mS/cm -> practical salinity. Quadratic fit to PSS-78, valid 8.96-59.73 mS/cm.
// Max error 0.058 ppt, versus 1.28 ppt for the common S = 0.66*EC linear rule.
float e = ec25 / 1000.0f;
float s_ppt = -0.47367528f + 0.59157002f*e + 0.0014494695f*e*e;
```

Two traps if you ever use that path: DFRobot's published cubic **already ends in `*0.5`** in some versions to give ppm, so people divide by 0.5 again and double-count; and the TDS-to-EC factor is ~0.7 for brackish water, not the 0.5 the library hard-codes. Skip ppm entirely — take EC straight from the cubic.

### 2.3 Turbidity

```c
// Generic/DFRobot optical turbidity. V reconstructed to the 5V domain after the divider.
// Response is INVERTED: clear water ~4.1V, dirty water lower.
float turbidity_ntu(float v5, bool *saturated)
{
    *saturated = false;
    if (v5 >= 4.2f)  return 0.0f;                       // clear; quadratic goes negative above this
    if (v5 <= 2.5f) { *saturated = true; return 3000.0f; }  // past the vertex, DO NOT TRUST
    return -1120.4f*v5*v5 + 5742.3f*v5 - 4352.9f;
}
```

**The clamp is not optional.** The quadratic has its vertex at V = 2.5626 V (3005 NTU) and then **turns around** — at 1.0 V it returns 269 NTU, a plausible-looking mid-range number for water that is actually opaque. Very dirty water silently reads clean. This is the single most likely runtime bug in the whole sensor stack.

**Two honesty notes for the report.** First, this quadratic is **not** DFRobot's official calibration — their sample code prints raw voltage only, and their FAQ says the part is qualitative and advises against converting to NTU. Second, the published clear-water spec is 4.1 +/- 0.3 V across 10-50 degC, which at ~2000 NTU/V is **+/- 600 NTU of thermal drift**. That dominates every other error term.

So: call the output a **relative turbidity index**, shroud the probe against ambient light (it is a bare IR LED/photodiode pair), log temperature alongside every reading, and show a monotonic response curve made by adding milk drop by drop. A stated limitation scores higher than a fake NTU figure.

### 2.4 Depth — MPX5010DP (or pot)

```c
// Datasheet: Vout = Vs * (0.09*P + 0.04), P in kPa. Ratiometric on Vs.
float mpx5010_depth_m(float vout, float vs, float salinity_ppt, float temp_c)
{
    float p_kpa = (vout/vs - 0.04f) / 0.09f;
    float rho   = 1000.0f + 0.77f*salinity_ppt - 0.0035f*(temp_c-15.0f)*(temp_c-15.0f);
    return (p_kpa * 1000.0f) / (rho * 9.80665f);
}
```

Full scale is **1.02 m of water** — fine for a bench demo, saturates instantly in the field. Resolution is 0.37 mm per count, but the datasheet accuracy band is +/-0.5 kPa = **+/-5.1 cm**, so do not report sub-millimetre depth.

**Three traps.** (1) The `DP` suffix is *dual-port differential*: the reference port must be **vented to atmosphere**. Seal it inside your enclosure and the pod becomes a barometer that drifts tens of centimetres with the weather. (2) `P` is in kPa, hydrostatic head needs Pa — the missing x1000 gives millimetres where you meant metres. (3) A 2:1 divider wastes 32% of ADC range; ~1.45:1 (10k/22k) recovers it.

Since depth barely affects absorption below 50 m (see `physics-models.md` §1), this channel is also a legitimate pot if the budget is tight.

---

## Stage 3 — Conditioning

Raw readings jitter. Jitter in the inputs becomes jitter in the waveform, and a parameter twitching on the judges' scope looks like a bug even when it is physics.

```c
typedef struct {
    float t_c, s_ppt, ntu, depth_m;
    float altitude_m;            // operator-set or assumed; see physics-models.md
    bool  turbidity_saturated;
    bool  valid;
} env_t;
```

Three filters, in this order:

1. **Median-of-5** per channel. Kills single-sample ADC spikes, which a mean would smear instead.
2. **Exponential moving average**, alpha ~0.2 at 10 Hz (about 0.5 s settling). Smooths without a buffer.
3. **Hysteresis on the decision, not the reading.** Re-solve only when an input moves more than a deadband: 0.5 degC, 1 ppt, 10% relative NTU, 0.2 m. Otherwise the waveform parameters chatter between two near-equal solutions.

```c
if (fabsf(env.t_c - last.t_c)   > 0.5f  ||
    fabsf(env.s_ppt - last.s_ppt) > 1.0f  ||
    fabsf(env.ntu - last.ntu) > 0.1f*last.ntu ||
    fabsf(env.depth_m - last.depth_m) > 0.2f) { resolve = true; }
```

Also range-check every channel and set `valid=false` on anything impossible (T outside -2..40, S outside 0..45, negative depth). An invalid channel falls back to its last good value and lights an LED — never to a default that silently changes the waveform.

---

## Stage 4 — Physics

Full derivations, coefficients and validity ranges are in `physics-models.md`. Implementation form only here.

```c
typedef struct { float c, alpha_sea, alpha_sed, alpha_tot, tl, nl, snr; } acoustics_t;

// Mackenzie 1981. T degC, S ppt, D metres -> m/s
float sound_speed(float T, float S, float D)
{
    float dS = S - 35.0f;
    return 1448.96f + 4.591f*T - 5.304e-2f*T*T + 2.374e-4f*T*T*T
         + 1.340f*dS + 1.630e-2f*D + 1.675e-7f*D*D
         - 1.025e-2f*T*dS - 7.139e-13f*T*D*D*D;
}

// Ainslie & McColm 1998. f kHz, T degC, S ppt, z KILOMETRES -> dB/km
// Boric acid term dropped: <0.6% of total above 10 kHz.
// Depth exponentials dropped: 0.08% difference between 5 m and 10 m.
float absorption_db_km(float f, float T, float S)
{
    float f2 = 42.0f * expf(T/17.0f);
    float mg = 0.52f * (1.0f + T/43.0f) * (S/35.0f) * (f2*f*f)/(f*f + f2*f2);
    float h2o = 0.00049f * f*f * expf(-T/27.0f);
    return mg + h2o;
}

// Urick viscous term, dominant for fine mud. d metres, Ms kg/m3 -> dB/km
float sediment_db_km(float f_hz, float c, float d, float Ms)
{
    const float nu = 1.0e-6f, rho_s = 2650.0f, rho_w = 1025.0f;
    float gamma = sqrtf((float)M_PI * f_hz / nu);
    float gd = gamma * d;
    float s  = (9.0f/(2.0f*gd)) * (1.0f + 2.0f/gd);
    float Tu = 0.5f + 9.0f/(2.0f*gd);
    float sg = rho_s/rho_w;
    float k  = 2.0f*(float)M_PI*f_hz/c;
    float zeta = (k/(2.0f*rho_s)) * (sg-1.0f)*(sg-1.0f) * (s/(s*s + (sg+Tu)*(sg+Tu)));
    return zeta * Ms * 8686.0f;      // Np/m -> dB/km
}
```

Supporting conversions and the link budget:

```c
float Ms = K_NTU * ntu / 1000.0f;               // K_NTU ~1-2 mg/L per NTU; STATE THE ASSUMPTION
float tl = 20.0f*log10f(R) + alpha_tot*R/1000.0f;          // single-way, dB
float nl = -15.0f + 20.0f*log10f(f_khz) + 10.0f*log10f(B); // thermal floor, dB re 1uPa
float gp = 10.0f*log10f(B * tau);                          // pulse compression gain
float snr = SL - 2.0f*tl + TS - nl + DI + gp;
```

**Cost: about 20 floating-point operations plus four transcendentals, at 10 Hz.** On a 100 MHz MCU that is roughly 0.001% CPU. It is not a power-budget item and it does not belong anywhere near the sample loop.

---

## Stage 5 — The decision

The physics is published. **This is the part you invent**, and it is what R-2.6 asks for.

**Objective: maximise resolution subject to meeting detection SNR at the required range, then minimise energy.**

```c
void decide(const env_t *e, const acoustics_t *a, wave_params_t *w)
{
    static const float FC_LADDER[] = {500e3f, 400e3f, 300e3f, 200e3f, 150e3f, 100e3f};
    float R = 7.0f * e->altitude_m;       // side-scan practice: fly at 10-20% of range scale

    // STEP 1: highest frequency whose link budget closes. R-2.7
    float fc = 100e3f;
    for (int i = 0; i < 6; i++) {
        float B = BETA_FRAC * FC_LADDER[i];
        if (link_budget_snr(FC_LADDER[i], B, TAU_NOMINAL, e, R) >= DT + SNR_MARGIN) {
            fc = FC_LADDER[i];
            break;
        }
    }

    // STEP 2: bandwidth. BETA_FRAC = 0.10 operating, 0.25 transducer ceiling. R-2.7
    w->bandwidth = BETA_FRAC * fc;
    w->f_center  = fc;
    w->f_start   = fc - w->bandwidth*0.5f;
    w->f_stop    = fc + w->bandwidth*0.5f;

    // STEP 3: pulse duration. R-2.8
    float bt_needed = powf(10.0f, (DT + SNR_MARGIN - snr_without_gp)/10.0f);
    w->tau = bt_needed / w->bandwidth;
    w->tau = clampf(w->tau, 1.0f/w->bandwidth,            // cannot beat inverse bandwidth
                            2.0f*R_MIN/a->c);             // blind range
    if (w->tau * PRF_HZ > DUTY_MAX) w->tau = DUTY_MAX / PRF_HZ;   // battery, R-0.8

    // STEP 4: amplitude. R-2.9. MINIMUM meeting margin, not maximum.
    w->amplitude = min_amplitude_for_margin(...);
}
```

**Step 4 is the non-obvious one and worth defending out loud.** In the reverberation-limited case — which is the normal side-scan case — source level and transmission loss **cancel** in the SNR expression. Shouting louder does not improve a seafloor image; it only drains the battery. So amplitude is set to the minimum that clears the margin. That is a physics-derived rule, and it is a much better answer than *"muddy water, so turn it up."*

Keep a **precomputed lookup table** as a stage fallback, generated offline from this same solver so the two can never disagree.

---

## Stage 6 — The complete waveform parameter set

Every feature of the wave, in one struct. This is the full contract between the slow layer and the fast layer.

```c
typedef enum { MOD_LFM, MOD_GEOMETRIC, MOD_PHASE_CODED } mod_type_t;
typedef enum { WIN_RECT, WIN_HANN, WIN_HAMMING, WIN_BLACKMAN } window_t;

typedef struct {
    /* --- identity --------------------------------------------------- */
    mod_type_t mod_type;        // R-1.7 : switchable at runtime
    uint32_t   seq;             // increments on every change; proves handover happened

    /* --- frequency (R-2.7) ------------------------------------------ */
    float    f_start;           // Hz, sweep start
    float    f_stop;            // Hz, sweep end
    float    f_center;          // Hz, (f_start+f_stop)/2
    float    bandwidth;         // Hz, f_stop-f_start
    int8_t   sweep_dir;         // +1 up-chirp, -1 down-chirp
    float    sweep_rate;        // Hz/s, = bandwidth/tau (LFM only)
    float    geo_ratio;         // f_stop/f_start (geometric only)

    /* --- time (R-2.8) ----------------------------------------------- */
    float    tau;               // s, pulse duration
    uint32_t n_samples;         // = tau * fs, rounded to buffer granularity
    float    fs;                // Hz, DAC sample rate
    float    pri;               // s, pulse repetition interval (ours to choose, not in PS)
    float    duty_cycle;        // tau/pri, the battery number

    /* --- amplitude (R-2.9) ------------------------------------------ */
    float    amplitude;         // 0.0-1.0 of DAC full scale
    uint16_t dac_offset;        // mid-scale bias, DAC is unipolar
    uint16_t dac_full_scale;    // 4095 for 12-bit

    /* --- phase ------------------------------------------------------ */
    float    phi0;              // rad, initial phase
    uint16_t phase_code;        // Barker-13 = 0x1F35, LSB-first
    uint8_t  n_chips;           // 13 for Barker-13
    uint32_t samples_per_chip;  // n_samples / n_chips

    /* --- shaping (R-3.5) -------------------------------------------- */
    window_t window;            // Hann / Hamming / Blackman / none

    /* --- derived, for display and validation ------------------------ */
    float    bt_product;        // bandwidth * tau
    float    proc_gain_db;      // 10*log10(bt_product)
    float    range_res_m;       // c / (2*bandwidth)
    float    max_range_m;       // from the link budget that chose this
} wave_params_t;
```

Everything the analog output can be is in that struct. **If a parameter is not in there, the waveform cannot express it** — which is a useful completeness check to run against R-1.8/1.9/1.10 and R-2.7/2.8/2.9 before the judging table.

---

## Stage 7 — Synthesis

### 7.1 The maths

**LFM chirp** (R-1.8). Frequency linear in time, phase therefore quadratic — this is the "complex trigonometric wave values" R-0.12 refers to:
```
k    = B / tau                                  sweep rate, Hz/s
f(t) = f0 + k*t
phi(t) = 2*pi*(f0*t + k*t^2/2) + phi0
s(t) = A * w(t) * sin(phi(t))
```

**Geometric sweep** (R-1.9). Frequency multiplies rather than adds:
```
r      = f1/f0
f(t)   = f0 * r^(t/tau)
phi(t) = 2*pi*f0*tau/ln(r) * (r^(t/tau) - 1) + phi0
```

**Phase-coded** (R-1.10). Fixed carrier, phase flipped per chip:
```
i      = floor(t / T_chip)
s(t)   = A * w(t) * sin(2*pi*fc*t + pi*c_i)      c_i in {0,1}
Barker-13 = + + + + + - - + + - + - +            peak autocorrelation sidelobe -22.3 dB
```

### 7.2 What the code actually does

Do not call `sinf()` per sample. All three reduce to **a phase accumulator plus one LUT read**, which is the hardware-level optimisation R-0.9 is asking for.

```c
#define LUT_BITS 12
#define LUT_SIZE (1u << LUT_BITS)       // 4096-entry quarter-wave or full sine, Q15
static int16_t sine_lut[LUT_SIZE];

static uint32_t phase_acc;              // Q32 phase, wraps naturally at 2*pi
static uint32_t phase_inc;              // Q32 increment = f/fs * 2^32
static int32_t  phase_inc_delta;        // LFM: per-sample change in phase_inc
static uint32_t geo_mult_q16;           // geometric: per-sample ratio, Q16
```

**LFM needs two accumulators, not one.** Frequency ramps linearly, so the *increment* itself ramps linearly:
```c
phase_inc0      = (uint32_t)((f_start / fs) * 4294967296.0);
phase_inc_delta = (int32_t)(((B/tau) / (fs*fs)) * 4294967296.0);

// per sample:
phase_acc += phase_inc;
phase_inc += phase_inc_delta;
```
Two adds per sample for a mathematically exact linear chirp. No multiply, no trig, no drift.

**Geometric** needs one multiply:
```c
geo_mult_q16 = (uint32_t)(powf(f_stop/f_start, 1.0f/n_samples) * 65536.0f);
// per sample:
phase_acc += phase_inc;
phase_inc  = (uint32_t)(((uint64_t)phase_inc * geo_mult_q16) >> 16);
```

**Phase-coded** is the cheapest — a constant increment, plus flipping the top phase bit on a chip boundary:
```c
phase_acc += phase_inc;
if ((n % samples_per_chip) == 0) {
    chip = (phase_code >> (n/samples_per_chip)) & 1;
}
uint32_t p = phase_acc + (chip ? 0x80000000u : 0u);   // +pi is just +2^31
```

One accumulator update, one table lookup, one window multiply. **Identical cost for all three modulation types** — which is exactly why "on the fly" switching (R-1.7) is free rather than expensive, and it is a good thing to be able to say.

---

## Stage 8 — Windowing (R-3.5)

Applied to the **amplitude envelope**, not the carrier. Multiplies each sample so the pulse fades in and out instead of slamming on.

```
n = 0 .. N-1
Hann      w[n] = 0.5  - 0.5*cos(2*pi*n/(N-1))
Hamming   w[n] = 0.54 - 0.46*cos(2*pi*n/(N-1))
Blackman  w[n] = 0.42 - 0.5*cos(2*pi*n/(N-1)) + 0.08*cos(4*pi*n/(N-1))
```

| Window | Peak spectral sidelobe | Main lobe |
|---|---|---|
| Rectangular (none) | -13 dB | narrowest |
| Hann | -31 dB | 2x |
| Hamming | -43 dB (nearest), slower far roll-off | 2x |
| Blackman | -58 dB | 3x |

These are **spectral** sidelobe figures — what the judges' FFT shows. Range sidelobes (ghost targets) are a receive-side matched-filter concern and out of scope. Do not claim the second.

Precompute the window into a Q15 table when `tau` changes (10 Hz, in the slow layer), never per sample:
```c
int16_t win_lut[MAX_SAMPLES];      // Q15
sample = ((int32_t)sine_lut[p >> (32-LUT_BITS)] * win_lut[n]) >> 15;
```

**The demo.** Capture the same pulse with `WIN_RECT` and with `WIN_BLACKMAN`, overlay the two FFTs, and report the measured sidelobe drop in dB. That single image is the most persuasive evidence you can produce for R-3.6, R-3.7 and R-3.8 together, and it takes ten minutes to make.

---

## Stage 9 — Buffer fill and DAC format

```c
#define BUF_SAMPLES 2048
static uint16_t dac_buf[2*BUF_SAMPLES];    // double buffer, DMA circular over both halves

void fill_half(uint16_t *dst, uint32_t n, const wave_params_t *w)
{
    for (uint32_t i = 0; i < n; i++) {
        phase_acc += phase_inc;
        if (w->mod_type == MOD_LFM) phase_inc += phase_inc_delta;
        // ... geometric / phase-coded variants
        int32_t s = ((int32_t)sine_lut[phase_acc >> (32-LUT_BITS)] * win_lut[samp_idx]) >> 15;
        s = (s * w->amplitude_q15) >> 15;
        dst[i] = (uint16_t)(w->dac_offset + (s >> 4));    // Q15 -> 12-bit unipolar
    }
}
```

The DAC is **unipolar**: a signed waveform must be biased to mid-scale (2048 on 12-bit) and the DC blocked later with a series capacitor, or handled by the amplifier stage. Forgetting the offset clips the entire negative half and produces a beautifully clean-looking half-wave that is completely wrong.

---

## Stage 10 — DMA streaming and parameter handover (RP2040 / PIO)

This is R-0.10, R-1.3, R-1.4 and R-1.6. **Platform: RP2040.** See `decisions/d1-platform-rp2040.md` for why.

```
DMA (DREQ_PIO0_TX0) -> PIO TX FIFO -> autopull -> OSR -> `out pins, N` -> GPIO -> external DAC
                                       ^
                              PIO clock divider (integer only)
```

### PIO is the output engine, not a timer + DAC peripheral

RP2040 has **no internal DAC of any kind**, so an external DAC is mandatory rather than a choice (this closes D-2's "internal or external" question from R-1.5). The output path is a PIO state machine shifting a sample to GPIO every clock.

A PIO state machine executes **one instruction per clock, always** — no pipeline hazards. Autopull refills the OSR from the FIFO in the same cycle at zero instruction cost. So a single-instruction loop sustains **one sample per clock = 125 MSPS** at stock clock.

Most parallel DACs latch on a clock edge, so you need to generate one with side-set:

```
.program dac_out
    out pins, 8   side 1
    nop           side 0
```

Two instructions per sample → **62.5 MSPS ceiling. At our 5 MSPS target we use 8% of PIO's capability.** Store 12-bit samples as `uint16` with the data in the low bits and `out pins, 16`, mapping the spare 4 GPIOs somewhere harmless — that keeps it at one `out` per sample.

### ⚠️ The clock divider must be an integer

`SMx_CLKDIV` is **16.8 fixed point** on RP2040 (1/256 steps — RP2350 is 16.16, don't follow guidance written for it). The divider does not synthesise a fractional clock: it gates a clock-enable off `sys_clk`, and with a fractional part it **alternates between N and N+1 system-clock periods** so only the long-term average is right.

Individual sample intervals then vary by a full system clock — **±8 ns at 125 MHz**. That is sample-clock jitter, and it lands directly on the judges' FFT as spurs and a raised noise floor. It would damage R-4.2 ("clean, low-distortion") for no reason.

**Use `div_frac = 0`, always.** Clean rates from 125 MHz:

| div_int | Rate | | div_int | Rate |
|---|---|---|---|---|
| 1 | 125 MSPS | | **25** | **5.000 MSPS** ✓ |
| 2 | 62.5 MSPS | | 50 | 2.5 MSPS |
| 5 | 25 MSPS | | 125 | 1.000 MSPS |
| 20 | 6.25 MSPS | | 250 | 500 kSPS |

**Trap: 10 MSPS is not reachable cleanly from 125 MHz** (needs 12.5), nor is 8 MSPS (15.625). If you want 10 MSPS, change `sys_clk` to **120 MHz** — it gives 10 MSPS at N=12, 6 at N=20, 5 at N=24, 4 at N=30. A mild underclock, no voltage change, and a far friendlier base. Pick the system clock to make the sample rate an integer divisor.

### DMA: chaining, not half-transfer interrupts

**RP2040 has no half-transfer interrupt.** The STM32 mental model in most tutorials does not transfer. RP2040 has **12 DMA channels**, and gapless streaming is done with **hardware channel chaining**: when a channel's count hits zero it immediately triggers the channel named in `CTRL.CHAIN_TO`, with no CPU and no interrupt latency.

```c
channel_config_set_chain_to(&cfg_a, chan_b);
channel_config_set_chain_to(&cfg_b, chan_a);   // ping-pong; a channel cannot chain to itself
```

Pacing comes from **`DREQ_PIO0_TX0`** — the TX FIFO asserts DREQ when it has space and the DMA refills it. Rate accuracy comes from the PIO divider, not the DMA. (DMA pacing timers exist but are fractional, with the same jitter character as above — don't use them here.)

### Better idea: one DMA per ping, not continuous ping-pong

**Sonar is pulsed, not continuous** — and RP2040 has **264 kB of SRAM**. A full 1 ms pulse at 5 MSPS is 5 000 samples = **10 kB, under 4% of SRAM**. Even 20 000 samples is 40 kB.

So skip ping-pong entirely:

1. Between pings, the main loop computes the **entire pulse** — DDS, window, amplitude — into one buffer.
2. At ping time, fire **one DMA transfer** of the whole buffer.
3. CPU sleeps (`WFI`) for the transmit and for the rest of the PRI.

At 100 Hz PRF with τ = 1 ms the duty cycle is 10%, and buffer regeneration costs roughly 100k cycles ≈ 0.8 ms, comfortably inside the 9 ms gap. **This is simpler than double-buffering, has no seam artefacts at buffer boundaries, and makes the parameter handover trivial** — new parameters take effect on the next ping, by construction.

Handover latency is then exactly **one PRI**, which is honest, bounded and measurable. Put the number on a slide instead of the word "instant" (R-2.6).

Keep the buffer in the striped SRAM0–3 region so DMA reads don't contend with code fetch, and mark the hot fill routine `__not_in_flash_func` so you aren't paying XIP cache misses.

### Sample rate budget

| f_max | Nyquist floor | Practical (10×) | RP2040 status |
|---|---|---|---|
| 500 kHz | 1 MSPS | **5 MSPS** | div_int = 25 exactly. PIO at 8% load. **Comfortable** |

Unlike the STM32 path — where a ~1 MSPS internal DAC makes 500 kHz genuinely hard — RP2040 + PIO has the digital side solved with an order of magnitude to spare. **The binding constraint moves to the external DAC and the analog front end**, which is where it belongs.

## Stage 11 — Fixed point

R-0.12 asks for cheap trigonometry under real-time constraints. The answer is: there is no trigonometry at runtime.

| Quantity | Format | Note |
|---|---|---|
| Phase accumulator | uint32 (Q32) | Wraps at 2*pi for free — this is why 32-bit is chosen |
| Sine LUT | int16 Q15 | 4096 entries = 8 KB. Quarter-wave symmetry cuts it to 2 KB if RAM is tight |
| Window LUT | int16 Q15 | Rebuilt only when tau changes |
| Amplitude | int16 Q15 | One multiply, one shift |
| Physics layer | float32 | 10 Hz, cost is irrelevant, keep it readable |

If the chosen STM32 has a **hardware CORDIC** peripheral, using it is a very quotable "hardware-level optimization" (R-0.9) — but verify your specific part has one before claiming it.

---

## Stage 12 — The ML/AI question

**Short answer: do not use ML in the decision path. YAGNI.** Being able to explain *why* is worth more marks than a neural network would be.

**Why it does not belong here:**

1. **The forward model is closed-form.** Ainslie-McColm plus Urick plus the sonar equation is about 20 floating-point operations with published, peer-reviewed coefficients. A learned surrogate would approximate a function you can already evaluate exactly — strictly worse, larger, and unexplainable.
2. **There is no training data.** No sea trials, no labelled environment-to-optimal-waveform pairs. Training on data generated by your own physics model teaches the network only to imitate the model, with added error.
3. **There is no feedback signal.** The PS is transmit-only. Without a receiver there is no echo quality, no reward, nothing to learn from. Reinforcement learning needs a loop that this system does not have.

**What to say if asked:** *"We considered a learned adaptation policy and rejected it. The forward model is analytic and costs twenty floating-point operations; a learned surrogate would be less accurate, larger, and harder to defend. We have no labelled field data and no receive-side feedback to train against. ML belongs on the receive side of this system — sediment classification from backscatter texture, target recognition in sonar imagery — which this problem statement excludes."*

That answer demonstrates judgment. "We added a neural network" invites the question of what it was trained on, which has no good answer.

**What you should build instead**, because these solve the real problems:

| Need | Use | Not |
|---|---|---|
| Noisy ADC causing parameter chatter | Median-of-5 + EMA + hysteresis (Stage 3) | A learned denoiser |
| Naming the regime on the display | Threshold rules over (T, S, NTU, depth) | A classifier |
| Solver too slow for the loop | It is not — 20 flops at 10 Hz | Model distillation |

The third row is the only one where ML-adjacent technique would ever be legitimate: if the physics solver were genuinely expensive, you would run it offline across the input space and compile the result into a lookup table or small decision tree. That is real engineering practice. **It is also unnecessary here by three orders of magnitude**, and saying exactly that — with the number — is a better slide than doing it anyway.

---

## Stage 13 — Verification hooks

Build these in from day one; they are your evidence for R-4.2, R-4.3 and I-4.

| Hook | Method | Proves |
|---|---|---|
| Commanded vs measured | Log every `wave_params_t`; compare against scope + FFT | R-4.2, R-4.3 |
| Golden reference | Regenerate the same waveform in Python from the same equations; overlay against scope CSV; report max error | Firmware maths matches textbook maths |
| CPU idle | Toggle a GPIO in the main loop, scope the duty cycle | R-0.11, I-4 |
| Handover latency | GPIO pulse on parameter swap, trigger scope on it | R-2.6 "instantly", with a number |
| Power | USB power meter, active vs sleep, energy per ping | R-0.8, I-3 |
| Window A/B | Same pulse, WIN_RECT vs WIN_BLACKMAN, overlaid FFTs | R-3.6, R-3.8 |

---

## Corrections to earlier documents

Recorded so nobody works from a stale version.

1. **SEN0244 TDS cannot measure seawater salinity** — saturates at 1.15 ppt against a 5-40 ppt requirement. Salinity is now a potentiometer channel. (`physics-models.md` §1.2 updated.)
2. **Sediment attenuation is viscous, not scattering** — Rayleigh f^4 applies to backscatter, a different quantity. (Fixed previously in `glossary.md`.)
3. **Fractional bandwidth is ~8% in real systems**, 20-35% transducer ceiling — not the 30-50% assumed earlier. (Fixed previously.)
4. **Use Steinhart-Hart, not Beta** — nameplate B=3950 costs 1.07 degC at 0 degC.
