# Telemetry Fault Detection — Algorithm & Risk Score Cheat Sheet

## 1. Overall Architecture

```text
Incoming Telemetry
        ↓
Per-device History
        ↓
Long Window (200) + Short Window (8)
        ↓
Median + MAD
        ↓
Anomaly Detection
   ┌────┴────┐
   ↓         ↓
Robust Z   Isolation Forest
   └────┬────┘
        ↓
     Anomaly
        ↓
Feature Extraction
        ↓
Fault-Specific Detection
        ↓
Final Failure Label + Risk Score
```

### Core idea

**Detection:** Is something wrong?

**Diagnosis:** What type of fault is it?

**Risk score:** How strongly does the observed behavior match that fault?

---

# 2. Long Window + Short Window

### Long window = 200 points

Used to learn the device's normal behavior.

```text
LongMedian
LongMAD
```

### Short window = 8 points

Used to understand what the device is doing **right now**.

```text
ShortMedian
ShortMAD
```

### Mental model

```text
Long window  → "What is normal?"
Short window → "What is happening now?"
```

---

# 3. Median + MAD — Robust Baseline

### Median

The middle value of the historical readings.

**Purpose:** Find the normal center without being strongly affected by outliers.

### MAD — Median Absolute Deviation

```text
MAD = median(|x - median(x)|)
```

**Purpose:** Measure normal sensor variation/noise.

### Why use it?

- Robust to outliers
- Good for noisy telemetry
- Creates an adaptive per-device baseline

**Keywords:** `median`, `MAD`, `robust statistics`, `baseline`, `outlier`

---

# 4. Robust Z-Score

```text
Robust Z =
|Current - LongMedian|
----------------------
       1.4826 × LongMAD
```

### Purpose

Measures how far the current reading is from normal behavior.

```text
Small score → probably normal
Large score → suspicious
```

### Example

```text
Current = 28
LongMedian = 25
LongMAD = 0.5

Robust Z = |28 - 25| / (1.4826 × 0.5)
         ≈ 4.05
```

**Keywords:** `robust Z-score`, `deviation`, `anomaly score`

---

# 5. Isolation Forest

### What is it?

An unsupervised ML algorithm for detecting unusual observations.

### Main idea

```text
Normal   → difficult to isolate
Anomaly  → isolated quickly
```

### Why use it?

Robust Z mainly looks at deviation from a baseline.

Isolation Forest can detect unusual **multivariate combinations** of features.

**Keywords:** `unsupervised learning`, `anomaly detection`, `decision trees`, `isolation`

---

# 6. Feature Extraction

Raw telemetry is converted into features before fault diagnosis.

### Temporal Features

Describe how the signal changes over time.

Examples:

- Median
- MAD
- Variance
- Rate of change
- Trend
- Autocorrelation

**Used for:** Spike, Drift, Flatline.

### Frequency Features

Describe periodic behavior.

Examples:

- Dominant frequency
- Spectral power
- Spectral entropy

**Used for:** Oscillation.

### Cross-channel Features

Describe relationships between sensors.

Examples:

- Correlation
- Difference
- Ratio
- Correlation change

**Used for:** Sensor Swap.

---

# 7. SPIKE — Hampel + Robust Z

### Pattern

A spike is a sudden abnormal value that returns toward normal.

```text
25 → 25 → 43 → 25 → 26
             ↑
           SPIKE
```

### Risk Score

```text
SpikeScore =
|Current - LongMedian|
----------------------
       LongMAD
```

Example:

```text
Current = 43
LongMedian = 25
LongMAD = 4

SpikeScore = |43 - 25| / 4
            = 4.5
```

### Trigger

```text
SpikeScore ≥ 4.5
+
recent values return toward normal
        ↓
      SPIKE
```

### Why Hampel?

Hampel uses a **local median + local MAD**, so the outlier does not strongly distort its own reference.

**Keywords:** `Hampel`, `local median`, `local MAD`, `isolated outlier`

---

# 8. DRIFT — Mann-Kendall + Theil-Sen

### Pattern

A drift is a persistent movement away from the normal level.

```text
25 → 26 → 27 → 29 → 31 → 33
     ↗    ↗    ↗    ↗
       persistent trend
```

### Risk Score

```text
DriftScore =
|ShortMedian - LongMedian|
--------------------------
        LongMAD
```

Example:

```text
ShortMedian = 29.5
LongMedian = 25
LongMAD = 0.5

DriftScore = |29.5 - 25| / 0.5
           = 9
```

### Additional trend check

Use Pearson correlation with time:

```text
r ≥ 0.6
```

A high positive/negative correlation indicates a consistent directional movement.

### Trigger

```text
DriftScore ≥ 3.0
+
|r| ≥ 0.6
+
trend persists
        ↓
      DRIFT
```

### Why Mann-Kendall + Theil-Sen?

- **Mann-Kendall:** Is there a statistically significant trend?
- **Theil-Sen:** What is the robust trend slope?

They are less dependent on normal-distribution assumptions than ordinary linear regression.

**Keywords:** `trend`, `non-parametric`, `Mann-Kendall`, `Theil-Sen`, `slope`, `p-value`

---

# 9. FLATLINE — Variance / Entropy + CUSUM

### Pattern

The sensor stops varying.

```text
Normal:
25.1 → 25.4 → 24.9 → 25.2 → 25.0

Flatline:
25.0 → 25.0 → 25.0 → 25.0 → 25.0
```

### Risk / condition

The important measurement is the **collapse in short-term variation**.

```text
MADshort ≤ 0.15 × MADlong
```

Example:

```text
MADlong = 0.5
MADshort = 0.04

0.15 × 0.5 = 0.075

0.04 ≤ 0.075
      ↓
FLATLINE candidate
```

### Variance

```text
Normal → noticeable variance
Flatline → variance ≈ 0
```

### Entropy

Shannon entropy:

```text
H(X) = -Σ p(x) log₂ p(x)
```

A flatline has almost one repeated value:

```text
p(50) = 1
H = 0
```

So:

```text
Variation ↓
Entropy ↓
        ↓
Possible flatline
```

### CUSUM

CUSUM detects a **persistent statistical change** rather than a single abnormal point.

For an upward shift:

```text
Sₜ⁺ = max(0, Sₜ₋₁⁺ + xₜ - μ₀ - k)
```

Alarm when:

```text
Sₜ⁺ > H
```

`H` is the decision threshold and should be calibrated from normal telemetry and validated using known faults.

### Trigger

```text
MADshort very small
+
variance/entropy reduced
+
change persists
        ↓
     FLATLINE
```

**Keywords:** `variance`, `entropy`, `Shannon entropy`, `CUSUM`, `persistence`

---

# 10. OSCILLATION — FFT + Autocorrelation

### Pattern

The signal repeatedly moves up and down.

```text
     /\      /\      /\
    /  \    /  \    /  \
___/    \__/    \__/    \___
```

### Amplitude Score

```text
OscillationScore =
MADshort
----------
MADlong
```

Example:

```text
MADshort = 1.0
MADlong = 0.5

Score = 1.0 / 0.5
      = 2.0
```

### Trigger

```text
OscillationScore ≥ 1.6
+
zero-crossing / direction-change rate ≥ 70%
        ↓
   OSCILLATION
```

### FFT

FFT converts:

```text
Time domain → Frequency domain
```

It identifies strong periodic frequencies.

```text
Strong frequency peak
        ↓
Periodic behavior
```

### Autocorrelation

Autocorrelation asks:

> "How similar is this signal to itself after a time shift?"

A strong peak at a particular lag means the pattern repeats.

```text
FFT
 ↓
"What frequency exists?"

Autocorrelation
 ↓
"Does the pattern repeat?"
```

Together they provide stronger evidence for oscillation.

**Keywords:** `FFT`, `frequency domain`, `dominant frequency`, `spectral power`, `autocorrelation`, `periodicity`, `lag`

---

# 11. SENSOR SWAP — Change-Point + Correlation

### Pattern

A sudden change occurs and the signal remains at a new stable level.

```text
25 → 25 → 25 → 45 → 45 → 45 → 45
                 ↑
            CHANGE POINT
```

### Risk Score

```text
SwapScore =
|ShortMedian - LongMedian|
--------------------------
        LongMAD
```

Example:

```text
ShortMedian = 45
LongMedian = 25
LongMAD = 0.5

SwapScore = |45 - 25| / 0.5
           = 40
```

### Trigger

```text
SwapScore ≥ 6.0
+
MADshort ≤ 0.6 × MADlong
+
change is persistent
        ↓
   SENSOR SWAP
```

### Change-point detection

Finds when the statistical behavior of the signal changes.

Possible methods:

- CUSUM
- PELT
- Bayesian Online Change Point Detection

### Cross-channel correlation

Checks whether the sensor still has the expected relationship with other channels.

```text
Sensor A → normal
Sensor B → normal
Sensor C → sudden permanent change
             ↓
      correlation changes
             ↓
      possible sensor swap
```

**Keywords:** `change point`, `distribution shift`, `correlation`, `cross-channel`, `sensor mapping`

---

# 12. How the Risk Score Works

The important point:

> **There is not one identical formula for every fault.**

Each fault has its own mathematical signature.

```text
SPIKE
↓
Distance from normal
↓
SpikeScore

DRIFT
↓
Short-term level shifted from normal
+
persistent trend
↓
DriftScore

FLATLINE
↓
Variation disappears
↓
MAD / variance / entropy

OSCILLATION
↓
Variation increases
+
periodicity
↓
OscillationScore

SENSOR SWAP
↓
Large permanent level shift
+
new level is stable
↓
SwapScore
```

---

# 13. Final Device Risk Score

Suppose the device has four channels:

```text
Temperature
Voltage
Vibration
Humidity
```

Calculate the relevant anomaly/fault scores for every channel.

Example:

| Channel | Strongest Score | Detected Fault |
|---|---:|---|
| Temperature | 1.2 | Normal |
| Voltage | 2.1 | Normal |
| Vibration | **5.7** | **Spike** |
| Humidity | 0.8 | Normal |

Then:

```text
FinalRiskScore =
max(Temperature, Voltage, Vibration, Humidity)

= max(1.2, 2.1, 5.7, 0.8)

= 5.7
```

Therefore:

```text
Final Failure Mode → SPIKE
Affected Channel   → Vibration
Final Risk Score   → 5.7
```

### Important

The `max()` operation is useful when the device's risk should be driven by its **most severe channel anomaly**.

---

# 14. Complete Risk-Scoring Flow

```text
                 RAW TELEMETRY
                       ↓
              Per-device history
                       ↓
              ┌────────┴────────┐
              ↓                 ↓
       Long Window (200)   Short Window (8)
              ↓                 ↓
        Long Median/MAD   Short Median/MAD
              └────────┬────────┘
                       ↓
             Robust anomaly score
                       +
                Isolation Forest
                       ↓
                "Something wrong"
                       ↓
              Feature extraction
                       ↓
       ┌───────────────┼────────────────┐
       ↓               ↓                ↓
    Temporal        Frequency       Cross-channel
    features         features         features
       ↓               ↓                ↓
   ┌──────────────────────────────────────┐
   │          FAULT CLASSIFIER            │
   │                                      │
   │ Spike       → Hampel + Robust Z      │
   │ Drift       → Mann-Kendall + Sen     │
   │ Flatline    → Variance/Entropy/CUSUM │
   │ Oscillation → FFT + Autocorrelation  │
   │ Sensor Swap → Change-point + Corr.   │
   └──────────────────┬───────────────────┘
                      ↓
             Fault-specific score
                      ↓
             Compare 4 channels
                      ↓
              MAX RISK SCORE
                      ↓
          FINAL FAILURE LABEL
```

---

# 15. Mulberry32 — Seed-Controlled Generator

## What is a PRNG?

**PRNG = Pseudo-Random Number Generator**

It generates numbers that look random but are determined by a starting value called a **seed**.

```text
Seed
 ↓
PRNG
 ↓
Pseudo-random sequence
```

## Mulberry32

Mulberry32 is a lightweight **32-bit deterministic PRNG**.

The key property:

```text
Same seed    → Same sequence
Different seed → Different sequence
```

Example:

```text
Seed = 1234
   ↓
0.21 → 0.87 → 0.43 → 0.65 → ...

Seed = 1234 again
   ↓
0.21 → 0.87 → 0.43 → 0.65 → ...
```

### Why use it?

It makes the telemetry/fault generator **repeatable and controllable**.

It can be used for:

- Device-target shuffling
- Fault sequence ordering
- Cycle offsets
- Reproducing demos/tests
- Re-randomizing when the seed changes

### Key idea

> **The seed controls the starting state; the PRNG deterministically generates everything that follows.**

**Keywords:** `PRNG`, `pseudo-random`, `seed`, `deterministic`, `Mulberry32`, `32-bit`, `reproducibility`

---

# 16. Quick Algorithm Memory Table

| Algorithm | Remember it as |
|---|---|
| **Median** | Normal center |
| **MAD** | Normal spread |
| **Robust Z-score** | How far is this point? |
| **Isolation Forest** | Is this observation unusual? |
| **Hampel** | Find isolated spikes |
| **Mann-Kendall** | Is there a trend? |
| **Theil-Sen** | How strong is the trend? |
| **Variance** | How much does it vary? |
| **Entropy** | How complex/unpredictable is it? |
| **CUSUM** | Did a change persist? |
| **FFT** | What frequency is present? |
| **Autocorrelation** | Does the pattern repeat? |
| **Change-point** | When did behavior change? |
| **Correlation** | Do sensors still relate? |
| **Mulberry32** | Same seed → same random sequence |

---

# 17. The Mental Model

Remember this chain:

```text
MEDIAN + MAD
    ↓
"What is normal?"
    ↓
ROBUST Z + ISOLATION FOREST
    ↓
"Is something wrong?"
    ↓
FEATURE EXTRACTION
    ↓
"What pattern is wrong?"
    ↓
FAULT-SPECIFIC ALGORITHM
    ↓
"Why is it wrong?"
    ↓
RISK SCORE
    ↓
FINAL FAILURE LABEL
```

### One-line revision

```text
Median/MAD       → Learn normal behavior
Robust Z-score   → Measure deviation
Isolation Forest → Find unusual patterns
Hampel           → Detect spikes
Mann-Kendall     → Detect trends
Theil-Sen        → Measure trend slope
Variance         → Detect loss of variation
Entropy          → Measure complexity
CUSUM            → Detect persistent change
FFT              → Detect frequencies
Autocorrelation  → Confirm periodicity
Change-point     → Find abrupt changes
Correlation      → Check sensor relationships
Mulberry32       → Generate deterministic random sequences
```

---

## Important Project Note

This is an **advanced architecture/cheat sheet**.

Do not claim that every algorithm above is implemented in the current repository unless it has actually been implemented and tested.

If the current implementation only contains:

```text
Median + MAD
+
Isolation Forest
```

then describe the remaining fault-specific algorithms as the **diagnosis/proposed layer** rather than claiming they are already running.


Here’s the **simple way to understand this whole risk-score system**:

### Big picture

For every new sensor reading, the system asks:

> **“How unusual is this reading compared with what this device normally does?”**

It keeps **two windows**:

* **Long window = 200 points** → learns the device's normal behavior.
* **Short window = 8 points** → looks at what the device is doing *right now*.

The **MAD** (Median Absolute Deviation) is basically a robust measure of **normal sensor variation/noise**.

So, generally:

> **Risk Score = “How many times bigger than normal is this abnormal behavior?”**

---

### 1. Spike

A spike is a **sudden abnormal reading that goes back to normal**.

Example:

```text
Normal:  25  25  26  25  25
Spike:                  43
Then:                         25  26
```

The formula:

$$
Score = \frac{|Current - LongTermMedian|}{LongTermMAD}
$$

If:

$$
Score \ge 4.5
$$

and the recent history is still near normal → **Spike**.

**Simple meaning:**

> “The current value is 4.5+ normal deviations away from what this sensor usually does.”

---

### 2. Drift

Drift is different because the sensor **gradually moves away from normal**.

```text
25 → 26 → 27 → 29 → 31 → 33
```

The system compares the recent 8-point median against the long-term median:

$$
Score =
\frac{|ShortMedian-LongMedian|}
{LongMAD}
$$

It also checks:

$$
r \ge 0.6
$$

The Pearson correlation is basically checking:

> **“Are the values consistently moving in one direction?”**

If score ≥ 3.0 and correlation ≥ 0.6 → **Drift**.

**Simple meaning:**

> “The sensor isn't suddenly broken; it's steadily moving away from its normal level.”

---

### 3. Flatline

A flatline means the sensor has **stopped varying**.

Normally:

```text
25.1 → 25.4 → 24.9 → 25.2 → 25.0
```

Flatline:

```text
25.0 → 25.0 → 25.0 → 25.0 → 25.0
```

The key thing being measured is **MAD**.

Normally:

$$
MAD_{short} \approx MAD_{long}
$$

But during a flatline:

$$
MAD_{short} \ll MAD_{long}
$$

The trigger is:

$$
MAD_{short} \le 0.15 \times MAD_{long}
$$

So the system says:

> **“This sensor has almost completely stopped changing.”**

---

### 4. Oscillation

Oscillation means the sensor is **rapidly bouncing back and forth**.

For example:

```text
25 → 32 → 24 → 33 → 23 → 31 → 24
```

The system checks two things:

**Amplitude:**

$$
Score =
\frac{MAD_{short}}{MAD_{long}}
$$

and **zero-crossing rate ≥ 70%**.

Zero-crossing rate essentially asks:

> **“How frequently is the signal switching direction/sign?”**

If it's switching rapidly and:

$$
Score \ge 1.6
$$

→ **Oscillation**.

**Simple meaning:**

> “The sensor is behaving much more violently than normal and repeatedly changing direction.”

---

### 5. Sensor Swap

This is like a **sudden permanent jump**.

Spike:

```text
25 → 25 → 45 → 25 → 25
             ↑
          temporary
```

Sensor swap:

```text
25 → 25 → 45 → 45 → 45 → 45
             ↑
          permanent
```

The system looks for:

$$
Score =
\frac{|ShortMedian-LongMedian|}
{LongMAD}
$$

with:

$$
Score \ge 6.0
$$

and relatively low short-term variation:

$$
MAD_{short} \le 0.6 \times MAD_{long}
$$

So it means:

> **“The sensor suddenly moved to a completely different stable operating level.”**

---

# How all 4 sensors become ONE risk score

Your device has four channels:

```text
Temperature
Voltage
Vibration
Humidity
```

Every time a new point arrives, the system evaluates **all four**.

For example:

| Channel     | Risk score | Detected  |
| ----------- | ---------: | --------- |
| Temperature |        1.2 | Normal    |
| Voltage     |        2.1 | Normal    |
| Vibration   |    **5.7** | **Spike** |
| Humidity    |        0.8 | Normal    |

The system takes the maximum:

$$
FinalRiskScore =
\max(1.2,2.1,5.7,0.8)
$$

Therefore:

$$
\boxed{FinalRiskScore=5.7}
$$

And it records:

```text
Failure Mode: Spike
Affected Channel: Vibration
Risk Score: 5.7
```

### The key idea to remember for your presentation

> **The long window tells us what “normal” looks like, while the short window tells us what the sensor is doing right now. Each failure mode has a mathematical signature, and the system chooses the strongest anomaly across all four channels as the device's final risk score.**

That is the easiest way to explain why you **don't just use one generic anomaly formula**: a spike, drift, flatline, oscillation, and sensor swap have fundamentally different patterns, so each gets a score designed around its specific behavior.
