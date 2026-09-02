# Telemetry Fault Detection — Algorithm Cheat Sheet

## 1. Overall Architecture

```text
Incoming Telemetry
        ↓
Per-device Median + MAD
        ↓
Anomaly Detection
   ┌────┴────┐
   ↓         ↓
Robust Z   Isolation Forest
   └────┬────┘
        ↓
Feature Extraction
        ↓
Fault-Specific Detection
        ↓
Final Failure Label
```

### Core idea

**Detection:** Is something wrong?

**Diagnosis:** What type of failure is it?

---

## 2. Baseline — Median + MAD

### Median
The middle value of historical readings.

**Purpose:** Find the device's normal center without being strongly affected by outliers.

### MAD — Median Absolute Deviation

```text
MAD = median(|x - median(x)|)
```

**Purpose:** Measure normal variation robustly.

### Why use it?
- Robust to outliers
- Better than mean/std when telemetry is noisy
- Creates a per-device adaptive baseline

**Keywords:** `baseline`, `median`, `MAD`, `robust statistics`, `outlier`

---

## 3. Robust Z-Score

```text
Robust Z = (x - median) / (1.4826 × MAD)
```

### Purpose
Measures how far a new reading is from the device's normal behavior.

### Interpretation
```text
Small score → probably normal
Large score → suspicious/anomalous
```

**Keywords:** `robust Z-score`, `deviation`, `anomaly score`, `threshold`

---

## 4. Isolation Forest

### What is it?
An unsupervised ML algorithm for anomaly detection.

### Main idea
Anomalies are easier to **isolate** than normal observations.

```text
Normal → needs many splits
Anomaly → isolated quickly
```

### Why use it?
Robust Z-score mainly looks at deviation from a baseline. Isolation Forest can detect unusual **multivariate combinations** across features.

**Keywords:** `unsupervised learning`, `anomaly detection`, `decision trees`, `isolation`, `path length`

---

# 5. Feature Extraction

Before diagnosing the fault, convert raw telemetry into useful features.

### Temporal Features
Describe how a signal changes over time.

Examples:
- Mean / median
- Variance
- Rate of change
- Rolling statistics
- Autocorrelation
- Number of repeated values

**Used for:** spikes, drift, flatlines.

### Frequency Features
Describe periodic behavior.

Examples:
- Dominant frequency
- Spectral power
- Spectral entropy

**Used for:** oscillation.

### Cross-Channel Features
Describe relationships between sensors.

Examples:
- Correlation
- Difference
- Ratio
- Change in correlation

**Used for:** sensor swap / channel inconsistency.

---

# 6. Fault-Specific Algorithms

## A. Spike → Hampel Filter + Robust Z

### What is a spike?
A sudden isolated abnormal reading.

```text
50 → 51 → 50 → 87 → 51 → 50
              ↑
            SPIKE
```

### Hampel Filter
Compares a point with its **local median + MAD**.

### Why use it?
- Detects isolated outliers
- Robust to noise
- Fast
- Works well online

**Keywords:** `local median`, `local MAD`, `outlier`, `Hampel`

---

## B. Drift → Mann-Kendall + Theil-Sen

### What is drift?
A slow, persistent movement away from normal.

```text
50 → 51 → 53 → 55 → 57 → 60
     ↗    ↗    ↗    ↗
       persistent trend
```

### Mann-Kendall
Asks:

> Is there a statistically significant trend?

### Theil-Sen
Estimates:

> How strong/fast is that trend?

### Why use them?
They are robust to noise and do not require normally distributed data.

**Keywords:** `trend`, `non-parametric`, `significance`, `slope`, `p-value`, `Theil-Sen`

---

## C. Flatline → Variance / Entropy + CUSUM

### What is a flatline?
The sensor stops varying.

```text
Normal:
49 51 50 52 49 51

Flatline:
50 50 50 50 50 50
```

### Variance
Measures how much the signal changes.

```text
Variance ↓↓↓ → possible flatline
```

### Entropy
Measures signal complexity/randomness.

```text
Entropy ↓↓↓ → signal became too predictable
```

### CUSUM
Cumulative Sum detects persistent statistical changes.

### Why use them?
Together they identify both the **loss of variation** and **when the change started**.

**Keywords:** `variance`, `entropy`, `CUSUM`, `change`, `persistence`

---

## D. Oscillation → FFT + Autocorrelation

### What is oscillation?
A repeating/periodic waveform.

```text
    /\      /\      /___/  \___/  \___/  \___
```

### FFT — Fast Fourier Transform
Converts:

```text
Time domain → Frequency domain
```

It reveals which frequencies are strong.

### Strong dominant frequency
```text
Strong peak
    ↓
Periodic behavior
    ↓
OSCILLATION
```

### Autocorrelation
Checks whether the signal resembles a delayed version of itself.

### Why use both?
FFT finds the frequency; autocorrelation confirms repeating behavior.

**Keywords:** `FFT`, `frequency domain`, `spectral power`, `dominant frequency`, `autocorrelation`, `periodicity`

---

## E. Sensor Swap → Change-Point + Correlation

### What is sensor swap?
A sensor/channel suddenly starts behaving like a different sensor or moves to a new stable distribution.

```text
3.7  3.8  3.7  3.8 | 7.3  7.3  7.4  7.3
                    ↑
               CHANGE POINT
```

### Change-Point Detection
Finds when the statistical behavior of the signal changed.

Possible methods:
- CUSUM
- PELT
- Bayesian Online Change Point Detection

### Cross-Channel Correlation
Checks whether the sensor still has the expected relationship with other channels.

### Why use them?
A permanent step + broken sensor relationships can indicate a channel/sensor mapping problem.

**Keywords:** `change point`, `distribution shift`, `correlation`, `channel mapping`, `sensor swap`

---

# 7. Final Fault Classifier

| Fault | Main Algorithm | Signal Signature |
|---|---|---|
| **Spike** | Hampel + Robust Z | Sudden isolated deviation |
| **Drift** | Mann-Kendall + Theil-Sen | Persistent directional trend |
| **Flatline** | Variance/Entropy + CUSUM | Variation collapses |
| **Oscillation** | FFT + Autocorrelation | Strong periodic pattern |
| **Sensor Swap** | Change-point + Correlation | Abrupt permanent distribution change |

---

# 8. The Mental Model

Remember this simple chain:

```text
MEDIAN + MAD
    ↓
"What is normal for this device?"
    ↓
ROBUST Z + ISOLATION FOREST
    ↓
"Is something abnormal?"
    ↓
FEATURE EXTRACTION
    ↓
"What pattern does the anomaly have?"
    ↓
FAULT-SPECIFIC ALGORITHM
    ↓
"Why is it abnormal?"
    ↓
FINAL FAILURE LABEL
```

### One-line explanation for each algorithm

```text
Median/MAD       → Learn normal behavior
Robust Z-score   → Measure abnormality
Isolation Forest → Find unusual patterns
Hampel           → Detect spikes
Mann-Kendall     → Detect trends
Theil-Sen        → Measure trend slope
Variance         → Detect loss of variation
Entropy          → Measure signal complexity
CUSUM            → Detect persistent change
FFT              → Detect frequencies
Autocorrelation  → Confirm periodicity
Change-point     → Find abrupt behavior changes
Correlation      → Check sensor relationships
```

---

## 9. Important Project Note

This is an **advanced proposed architecture**.

Do not claim that the current repository implements every algorithm above unless you have actually implemented and tested them.

The core documented components are:

```text
Median + MAD
      +
Isolation Forest
      ↓
Anomaly Detection
```

The fault-specific algorithms can be presented as the **proposed diagnosis layer** if they are not yet implemented.

---

## 10. Latency — Judge Question

**Question:** How do you guarantee sub-200 ms latency at 500 events/second?

**Answer:**

> "Our ingestion layer uses asynchronous, non-blocking processing with lightweight anomaly calculations and buffered database/broadcast operations. WebSocket-based communication avoids HTTP polling, while frontend buffering minimizes unnecessary rendering. The target latency is validated through load testing rather than assumed."

**Keywords:** `async`, `non-blocking`, `buffering`, `WebSocket`, `batching`, `load testing`, `latency`


---

# 11. Seed-Controlled Generator — Mulberry32 PRNG

## What is a PRNG?

**PRNG = Pseudo-Random Number Generator**

It produces numbers that look random but are actually generated deterministically from a starting value called a **seed**.

```text
Seed
 ↓
PRNG
 ↓
Pseudo-random sequence
```

## Mulberry32

**Mulberry32** is a lightweight **32-bit deterministic PRNG**.

The important property is:

```text
Same seed → Same sequence
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

The generated sequence can be used for:

- Shuffling device targets
- Fault sequence ordering
- Cycle offsets
- Reproducing the same demo/test
- Re-randomizing when the seed changes

### Why not use normal `Math.random()`?

A seeded PRNG lets you reproduce exactly the same sequence.

That is useful for debugging and demonstrations:

```text
Seed = 42
      ↓
Same devices
Same fault ordering
Same offsets
Same generated sequence
```

Change the seed:

```text
Seed = 99
      ↓
New pseudo-random sequence
```

### Key idea

> **The seed controls the starting state; the PRNG deterministically generates everything that follows.**

**Keywords:** `PRNG`, `pseudo-random`, `seed`, `deterministic`, `Mulberry32`, `32-bit`, `reproducibility`, `random sequence`

---

# 12. Judge Question — Seed-Controlled Generator

**Question:** How does the seed control the generator?

**Answer:**

> "We use a Mulberry32 32-bit deterministic pseudo-random number generator. Given any seed, the PRNG produces the exact same sequence of pseudo-random numbers, which we use to shuffle device targets, determine fault sequence ordering, and generate cycle offsets. Changing the seed re-randomizes the sequence while keeping the generator deterministic and reproducible."

### Short version

> **"Same seed gives the same sequence; changing the seed gives a new reproducible sequence."**

---

# 13. Complete Algorithm Cheat Sheet

```text
MEDIAN + MAD
    ↓
Learn device's normal behavior

ROBUST Z-SCORE
    ↓
Measure how abnormal a reading is

ISOLATION FOREST
    ↓
Find unusual multivariate patterns

HAMPEL FILTER
    ↓
Detect isolated spikes

MANN-KENDALL
    ↓
Detect statistically significant trends

THEIL-SEN
    ↓
Measure robust trend slope

VARIANCE
    ↓
Detect loss of signal variation

ENTROPY
    ↓
Measure signal complexity

CUSUM
    ↓
Detect persistent statistical changes

FFT
    ↓
Detect frequency / periodic behavior

AUTOCORRELATION
    ↓
Confirm repeating patterns

CHANGE-POINT
    ↓
Find abrupt distribution changes

CORRELATION
    ↓
Check relationships between sensors

MULBERRY32 PRNG
    ↓
Generate deterministic pseudo-random sequences
```

## Final Mental Model

```text
                 TELEMETRY
                     ↓
              MEDIAN + MAD
                     ↓
              "What's normal?"
                     ↓
          ROBUST Z + ISOLATION FOREST
                     ↓
             "Is something wrong?"
                     ↓
             FEATURE EXTRACTION
                     ↓
              "What pattern?"
                     ↓
        ┌────────────┼────────────┐
        ↓            ↓            ↓
     TEMPORAL     FREQUENCY   CROSS-CHANNEL
        ↓            ↓            ↓
     Spike         FFT        Correlation
     Drift                    Change-point
     Flatline
        └────────────┼────────────┘
                     ↓
              FAULT CLASSIFIER
                     ↓
        SPIKE / DRIFT / FLATLINE /
        OSCILLATION / SENSOR SWAP
                     ↓
             FAILURE LABEL
```

---

## 14. One-Minute Revision

| Concept | Remember it as |
|---|---|
| **Median** | Normal center |
| **MAD** | Normal spread |
| **Robust Z** | How abnormal? |
| **Isolation Forest** | Isolate unusual data |
| **Hampel** | Find spikes |
| **Mann-Kendall** | Is there a trend? |
| **Theil-Sen** | How strong is the trend? |
| **Variance** | How much does it vary? |
| **Entropy** | How complex is it? |
| **CUSUM** | Did behavior persistently change? |
| **FFT** | What frequencies exist? |
| **Autocorrelation** | Does the pattern repeat? |
| **Change-point** | When did behavior change? |
| **Correlation** | Do sensors still relate? |
| **Mulberry32** | Same seed → same random sequence |

---

## 15. Important Project Note

This is an **advanced proposed architecture**.

Do not claim that the current repository implements every algorithm above unless you have actually implemented and tested them.

The core documented components are:

```text
Median + MAD
      +
Isolation Forest
      ↓
Anomaly Detection
```

The fault-specific algorithms can be presented as the **proposed diagnosis layer** if they are not yet implemented.

The Mulberry32 generator should likewise be described as implemented only if it actually exists in the generator code.
