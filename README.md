

````markdown
# Smart Radar Adaptive Scanning

### GRU-Based Temporal Prediction and Bayesian Adaptive Scan Scheduling

A research prototype for intelligent radar spectrum scanning using the **Turing Synthetic Radar Dataset (TSRD)**.

The system predicts emitter timing, estimates emitter state, calculates scan priority, predicts rendezvous windows, and adaptively selects the next scan using Thompson Sampling.

---

## 1. System Overview

```text
RAW INTERLEAVED PDWs
        ↓
PDW Association / Tracking
        ↓
Temporal Feature Extraction
        ↓
Timing GRU
        ↓
Bayesian State Estimation
        ↓
Priority / Risk Score
        ↓
Rendezvous Prediction
        ↓
Thompson Sampling
        ↓
Adaptive Scan
        ↓
HIT / MISS
        ↓
Bayesian Update
````

The core idea is:

> **Predict when an emitter is likely to appear, estimate its current state, and allocate limited scan opportunities accordingly.**

---

## 2. Dataset

The project uses:

**Turing Synthetic Radar Dataset (TSRD)**

Primary experiment:

```text
scan/train_scan/config_0.h5
```

| Property                   |   Value |
| -------------------------- | ------: |
| PDWs                       | 169,617 |
| Raw features               |       5 |
| Observed emitters          |      72 |
| Transmitter configurations |      96 |

### Raw Features

* Time of Arrival (ToA)
* Frequency
* Pulse Width
* Angle of Arrival (AoA)
* Amplitude

Emitter labels were used only for development-time tracking and evaluation, not as model input.

---

## 3. Feature Engineering

Seven temporal features were generated:

```text
log_delta_toa
frequency
pulse_width
sin_aoa
cos_aoa
amplitude
zero_delta_flag
```

### Timing Transformation

```text
log_delta_toa = log(1 + ΔToA)
```

This reduces the effect of the highly heavy-tailed timing distribution.

### AoA Encoding

```text
sin(AoA)
cos(AoA)
```

is used to avoid the discontinuity around ±180°.

---

## 4. Data Preparation

Each emitter track was split chronologically:

```text
70% Training
15% Validation
15% Test
```

The primary model uses a:

```text
32-PDW temporal window
```

Sequence shapes:

| Split      | Shape             |
| ---------- | ----------------- |
| Train      | `(116667, 32, 7)` |
| Validation | `(23646, 32, 7)`  |
| Test       | `(23673, 32, 7)`  |

---

# 5. Timing GRU

The final timing-focused GRU is:

```text
32 × 7 Input
     ↓
   GRU(64)
     ↓
 Dropout(0.20)
     ↓
 Dense(32, ReLU)
     ↓
 Dense(1)
     ↓
Predicted next ΔToA
```

### Configuration

| Parameter     |  Value |
| ------------- | -----: |
| GRU units     |     64 |
| Dropout       |   0.20 |
| Dense         |     32 |
| Optimizer     |   Adam |
| Learning rate |  0.001 |
| Loss          |  Huber |
| Epochs        |     30 |
| Parameters    | 16,129 |

---

# 6. GRU Results

## Timing-Only GRU — Validation

| Metric            |      Result |
| ----------------- | ----------: |
| Normalized MAE    |  **0.4102** |
| Physical ΔToA MAE | **2729.95** |
| Median error      |   **39.86** |
| P90 error         |  **261.27** |
| P95 error         |  **519.63** |

> **0.4102 is a validation result, not a held-out test result.**

---

## Held-Out Multi-Output GRU Test

The clean held-out test result available from the earlier GRU is:

| Metric          |       Result |
| --------------- | -----------: |
| Normalized MAE  | **0.425930** |
| Normalized RMSE | **0.633639** |
| Physical MAE    |  **3808.42** |
| Median error    |    **48.22** |
| P90 error       |   **522.58** |
| P95 error       |  **1204.54** |

---

# 7. GRU vs Statistical Baseline

A median-of-previous-32 timing predictor was used as the statistical baseline.

| Model       |          MAE |         RMSE |
| ----------- | -----------: | -----------: |
| Statistical |     0.459292 |     0.717426 |
| GRU         | **0.425930** | **0.633639** |

Improvement:

```text
MAE  ≈ 7.26%
RMSE ≈ 11.68%
```

Emitter-level evaluation:

```text
GRU better:        29 / 44 emitters
Statistical better: 15 / 44 emitters
```

---

# 8. Bayesian State Tracking

A Beta-Bernoulli model estimates emitter activity.

```text
P(active) = α / (α + β)
```

Example:

```text
Initial       → 0.50
After HIT     → 0.6667
After HIT     → 0.7500
After MISS    → 0.6000
```

This state is passed to the scheduling layer.

---

# 9. Priority / Risk Score

The prototype combines timing urgency, activity, confidence and uncertainty:

```text
TimingUrgency =
exp(-max(ΔToA, 0) / timing_scale)

Priority =
0.40 × TimingUrgency
+ 0.35 × P(active)
+ 0.20 × Confidence
- 0.05 × Uncertainty
```

Example:

| Emitter |   Priority |
| ------- | ---------: |
| A       | **0.8215** |
| B       |     0.5053 |
| C       |     0.2352 |

---

# 10. Rendezvous Prediction

The GRU prediction is converted into a future scan opportunity:

```text
Rendezvous Time =
Current ToA + Predicted ΔToA
```

An uncertainty-dependent scan window is then created.

Example:

```text
Current ToA = 1000
Predicted ΔToA = 20
Uncertainty = 2

Rendezvous = 1020
Window = 1015 – 1025
```

---

# 11. Thompson Sampling

Each emitter maintains a Beta distribution:

```text
θᵢ ~ Beta(αᵢ, βᵢ)
```

The scheduler combines priority and Thompson sampling:

```text
FinalScore =
0.70 × Priority
+ 0.30 × ThompsonSample
```

After each scan:

```text
HIT  → α + 1
MISS → β + 1
```

An exploration term was also tested:

```text
Exploration =
1 / sqrt(1 + scan_count)
```

---

# 12. Closed-Loop Simulation

A synthetic environment was created to test the complete feedback loop.

```text
Prediction
    ↓
Priority
    ↓
Scheduler
    ↓
Scan
    ↓
HIT / MISS
    ↓
Bayesian Update
    ↓
Next Decision
```

### Result

Over 100 simulated scans:

| Metric            |    Result |
| ----------------- | --------: |
| Scans             |       100 |
| Hits              |        91 |
| Misses            |         9 |
| Hit Rate          |   **91%** |
| Cumulative Reward | **88.75** |

> **Important:** The 91% hit rate is from a controlled synthetic simulation. It is **not TSRD accuracy or real-world radar detection accuracy**.

---

# 13. Technology Stack

### ML / Data

* Python
* TensorFlow / Keras
* GRU
* NumPy
* Pandas
* h5py
* HDF5
* Scikit-learn

### Probabilistic Scheduling

* Beta-Bernoulli Bayesian inference
* Thompson Sampling
* Uncertainty-aware rendezvous
* Adaptive exploration

### Visualization

Interactive UI / digital-twin layer for:

* emitter visualization
* frequency regions
* timing predictions
* scan scheduling
* scheduler decisions

---

# 14. Final Architecture

```text
Interleaved PDWs
      ↓
Emitter Tracking
      ↓
Temporal Features
      ↓
Timing GRU
      ↓
Bayesian State
      ↓
Priority Score
      ↓
Rendezvous Window
      ↓
Thompson Sampling
      ↓
Adaptive Scan
      ↓
Hit / Miss
      ↓
State Update
```

---

# 15. Limitations

* Current emitter-specific experiments use ground-truth labels for development tracking.
* The GRU predicts timing but does not independently solve full blind PDW deinterleaving.
* Extreme timing gaps create large MAE/RMSE values.
* The timing-only GRU's `0.4102` result is validation performance.
* Scheduler weights are prototype parameters.
* The 91% scheduler result is synthetic and not a TSRD benchmark.
* Real deployment requires receiver bandwidth, dwell time, retuning latency, scan cost and detection constraints.

---

# 16. Future Work

### 1. Real PDW Association

Replace ground-truth emitter IDs with actual:

```text
PRI / ToA
+ Frequency
+ PW
+ AoA
+ Amplitude
+ Track consistency
```

### 2. Joint Association + Prediction

Use predicted timing to assist emitter association.

### 3. Better Uncertainty Modeling

Explore:

* Monte Carlo Dropout
* GRU Ensembles
* Quantile Regression
* Probabilistic GRUs

### 4. Frequency-Aware Scheduling

Move from emitter-only scheduling to:

```text
Emitter + Frequency Region
```

under receiver bandwidth constraints.

### 5. Hardware-in-the-Loop

Connect:

```text
Prediction
→ Receiver Configuration
→ Scan
→ Detection
→ Bayesian Update
```

to an SDR/RF simulator.

---

# 17. Project Status

| Component                          | Status |
| ---------------------------------- | ------ |
| TSRD loading                       | ✅      |
| PDW processing                     | ✅      |
| Temporal features                  | ✅      |
| GRU prediction                     | ✅      |
| Statistical baseline               | ✅      |
| Bayesian tracking                  | ✅      |
| Priority scoring                   | ✅      |
| Rendezvous prediction              | ✅      |
| Thompson Sampling                  | ✅      |
| Synthetic closed-loop simulation   | ✅      |
| Interactive UI                     | ✅      |
| Real PDW association               | ⏳      |
| Full TSRD deinterleaving benchmark | ⏳      |
| Hardware validation                | ⏳      |

---

# 18. Key Takeaway

The project demonstrates a complete closed-loop adaptive radar scanning architecture:

```text
PREDICT
   ↓
ESTIMATE
   ↓
PRIORITIZE
   ↓
SCHEDULE
   ↓
SCAN
   ↓
LEARN
   ↺
```

The main experimental finding is that the GRU captures useful temporal structure beyond a simple statistical timing baseline, while the Bayesian and Thompson Sampling layers convert those predictions into an adaptive scan decision.

The next major step is replacing ground-truth emitter grouping with a real PDW association/deinterleaving system and evaluating the complete pipeline under realistic receiver constraints.

```

**This is the version I'd actually put on your GitHub.** It keeps the important numbers, formulas, architecture, limitations, and future work, but removes most of the explanatory repetition from the previous README.
```
