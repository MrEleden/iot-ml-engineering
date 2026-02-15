# Frequency-Domain Features

Spectral analysis of sensor signals — when it works, when it doesn't, and what's actually computable at 1 reading per minute.

## The Honest Limitation Up Front

Traditional frequency-domain analysis (vibration FFT for bearing fault detection, current spectrum analysis for motor diagnostics) requires sampling rates of hundreds to thousands of hertz. Our sensor data arrives at approximately **1 reading per minute** (~0.017 Hz). This is orders of magnitude too slow for classical vibration analysis.

The Nyquist theorem limits the maximum detectable frequency to half the sampling rate. At 1 sample/minute:

- **Maximum detectable frequency**: 0.5 cycles/minute = 0.0083 Hz
- **Minimum detectable period**: 2 minutes

This means we **cannot** detect:

- Bearing defect frequencies (typically 20–500 Hz)
- Gear mesh frequencies (hundreds of Hz)
- Motor electrical frequencies (50/60 Hz and harmonics)
- Rotor imbalance frequencies (typically 10–60 Hz)

These require dedicated high-frequency vibration sensors with their own data acquisition systems, which are outside [project scope](../05-architecture/system-overview.md).

## What Frequency Analysis CAN Detect at 1/min

Despite the severe sampling rate limitation, spectral analysis at 1/min can detect **slow periodic patterns** with periods of 2 minutes or longer. In refrigeration systems, several operationally significant behaviors fall in this range:

### Compressor Short-Cycling

**Period**: 2–15 minutes (well within our detection range)

A healthy compressor runs for 10–30+ minutes, shuts off for a rest period, then restarts. A failing compressor may cycle on and off every 2–5 minutes. This "short-cycling" pattern has a characteristic frequency of 0.07–0.5 cycles/minute — detectable at 1/min sampling.

Short-cycling shows up in `current_compressor` (oscillation between running current and zero) and `vibration_compressor` (oscillation between vibration and baseline). An FFT over a 1-hour window (60 samples) reveals whether the compressor is cycling at an abnormally high frequency.

### Defrost Cycle Regularity

**Period**: 4–12 hours (easily detectable)

Defrost cycles follow a programmed schedule — typically every 6–8 hours. A device whose defrost frequency shifts (more frequent = ice buildup problem, less frequent = controller fault) shows a changed spectral signature in the `defrost_cycle` boolean signal over a 1-day window.

This is better captured by `defrost_active_fraction` (see [Time-Domain](./time-domain.md)), but spectral analysis can confirm periodicity — an irregular defrost pattern (aperiodic rather than periodic) suggests a different failure mode than a shifted-but-regular pattern.

### Door Usage Patterns

**Period**: minutes to hours

The `door_open_close` signal may exhibit periodic patterns tied to operational schedules (shift changes, delivery windows). Spectral analysis can distinguish between regular-interval door openings (expected) and random or sustained openings (potential issue). This is a secondary use case — `door_open_fraction` is usually sufficient.

### Temperature Oscillations

**Period**: 2–30 minutes

Evaporator and condenser temperatures oscillate naturally as the compressor cycles. The dominant period of these oscillations correlates with compressor cycle time. A shift in the dominant oscillation period of `temperature_evaporator` from 20 minutes to 5 minutes indicates short-cycling, providing a cross-check with the current-based detection above.

## Spectral Features

Given the limitations above, the following spectral features are meaningful at 1/min sampling. These features are **not in the [Contract 3 schema](../05-architecture/data-contracts.md)** — they are candidates for future schema additions once validated against real anomaly labels.

### Spectral Entropy

**Definition**: the Shannon entropy of the normalized power spectrum. High entropy means the signal's energy is spread across many frequencies (noisy, unpredictable). Low entropy means the energy is concentrated in a few frequencies (periodic, predictable).

**Application**: a healthy compressor produces a low-entropy current signal (regular on/off cycling at a consistent frequency). A degrading compressor produces higher entropy (irregular cycling, unstable operation).

```python
# Conceptual Pandas UDF for spectral entropy — runs inside @transform_df
import numpy as np
from scipy.stats import entropy

def spectral_entropy(values: np.ndarray) -> float:
    """Compute spectral entropy of a 1D signal."""
    if len(values) < 4:
        return None  # Not enough data for meaningful FFT
    
    # Compute power spectrum via FFT
    fft_vals = np.fft.rfft(values - np.mean(values))  # Remove DC offset
    power = np.abs(fft_vals) ** 2
    
    # Normalize to probability distribution
    power_sum = np.sum(power)
    if power_sum == 0:
        return 0.0  # Constant signal
    
    power_norm = power / power_sum
    
    # Shannon entropy, normalized by log(N) to give range [0, 1]
    return float(entropy(power_norm) / np.log(len(power_norm)))
```

**Sensors**: compute on `current_compressor` and `vibration_compressor` within 1-hour windows (60 samples). At 15-minute windows (15 samples), spectral resolution is too coarse for reliable entropy estimation.

**Null behavior**: return null if fewer than 4 non-null readings exist in the window (FFT on 3 or fewer points is meaningless).

### Dominant Frequency

**Definition**: the frequency with the highest power in the spectrum, excluding DC (0 Hz).

**Application**: the dominant frequency of `current_compressor` in a 1-hour window corresponds to the compressor cycling rate. A shift from 0.05 cycles/min (20-minute cycle) to 0.2 cycles/min (5-minute cycle) indicates short-cycling. A shift toward lower frequency (longer cycles) might indicate a compressor struggling to reach setpoint.

```python
def dominant_frequency(values: np.ndarray, sample_rate_per_min: float = 1.0) -> float:
    """Return the dominant non-DC frequency in cycles per minute."""
    if len(values) < 4:
        return None
    
    fft_vals = np.fft.rfft(values - np.mean(values))
    power = np.abs(fft_vals) ** 2
    freqs = np.fft.rfftfreq(len(values), d=1.0 / sample_rate_per_min)
    
    # Exclude DC component (index 0)
    if len(power) < 2:
        return None
    
    dominant_idx = np.argmax(power[1:]) + 1
    return float(freqs[dominant_idx])
```

**Interpretation**: at 1 sample/min over a 60-sample window, the frequency resolution is 1/60 = 0.0167 cycles/min. This means we can distinguish between a 60-minute cycle and a 30-minute cycle, but not between a 5-minute and a 4.5-minute cycle. The resolution gets coarser for shorter windows.

### Spectral Energy Ratio

**Definition**: ratio of energy in a specific frequency band to total energy.

**Application**: define a "short-cycling band" (e.g., 0.1–0.5 cycles/min, corresponding to 2–10 minute cycles). The fraction of signal energy in this band serves as a short-cycling indicator. High ratio → compressor is spending most of its energy in rapid cycling.

This is more targeted than spectral entropy — it asks "how much short-cycling is happening?" rather than "how irregular is the signal overall?"

## Practical Considerations

### Minimum Window Size for FFT

FFT quality degrades rapidly with fewer samples. Guidelines for our sampling rate:

| Window | Samples | Frequency Resolution | Usable? |
|--------|---------|---------------------|---------|
| 15-min | ~15 | 0.067 cycles/min | Marginally — can detect presence of oscillation but not its frequency accurately |
| 1-hour | ~60 | 0.017 cycles/min | Yes — sufficient to characterize compressor cycling patterns |
| 1-day | ~1440 | 0.0007 cycles/min | Yes — can detect hourly and multi-hour periodicities (defrost cycles, load patterns) |

Spectral features should be computed on 1-hour and 1-day windows only. The 15-minute window does not provide enough samples for frequency resolution that adds value beyond what time-domain statistics already capture.

### Handling Missing Readings

FFT assumes evenly-spaced samples. Missing readings (gaps in the time series) introduce spectral artifacts — false frequencies that don't exist in the underlying signal. Options:

1. **Linear interpolation**: fill gaps by interpolating between adjacent readings. Acceptable for small gaps (1–2 missing readings). Larger gaps should set the spectral features to null for that window.
2. **Zero-fill**: replace missing readings with zero. This introduces discontinuities that leak energy across all frequencies — avoid.
3. **Skip the window**: if more than 20% of expected readings are missing, set spectral features to null. A spectral analysis on a signal with 30% gaps is unreliable.

Recommendation: interpolate gaps of ≤ 2 minutes, null-out spectral features for windows with > 20% missing readings.

### Compute Cost of FFT in Foundry

FFT is not a native PySpark function. It requires a Pandas UDF (or equivalent) to execute NumPy/SciPy functions per device per window. At 100K devices:

- **1-hour windows**: 100K devices × 24 windows/day = 2.4M FFT computations/day. Each FFT on 60 samples is trivially fast (microseconds), but the Pandas UDF serialization overhead adds up. Group devices into reasonably-sized partitions (1,000–10,000 devices per partition) to amortize UDF launch cost.
- **1-day windows**: 100K × 1 window/day = 100K FFTs/day. Negligible.

The bottleneck is not the FFT itself but the Arrow serialization between Spark and Pandas. Profile before optimizing — this may not be a bottleneck at all.

### Spectral Feature Drift Monitoring

Spectral features are sensitive to operational pattern changes that may not affect time-domain statistics:

**Seasonal shifts in compressor cycling**: compressor cycle frequency varies with thermal load. Summer heat forces more frequent cycling (shorter periods, higher dominant frequency), while mild weather allows longer cycles. A spectral entropy increase fleet-wide in spring is expected seasonal behavior, not model drift. Monitoring should compare spectral feature distributions against the same calendar month from the prior year, not just the prior week.

**Firmware update impacts**: firmware updates that change compressor control logic (e.g., adjusting minimum run time, changing setpoint deadband) directly alter the cycling pattern that spectral features measure. A firmware rollout to 10K devices will appear as abrupt spectral drift for those devices. The monitoring system must be able to filter by firmware version (available in the device registry via the [Ontology](../04-palantir/ontology-design.md)) to distinguish firmware-driven spectral changes from equipment degradation.

**Relationship to time-domain monitoring**: spectral drift often co-occurs with time-domain distribution shifts (e.g., a change in `current_compressor_std` accompanies a change in spectral entropy). Link spectral monitoring to the [time-domain feature distribution monitoring](./time-domain.md) to avoid duplicate alerting. When both time-domain and spectral drift are detected simultaneously, a single investigation covers both.

### Schema Implications

Spectral features are **not currently in the [`device_features` Contract 3 schema](../05-architecture/data-contracts.md)**. Before adding them:

1. Validate on a subset of devices that spectral features improve anomaly detection (measure false positive/negative rates with and without)
2. Add as nullable columns to Contract 3 — existing consumers are unaffected
3. Follow the [schema evolution strategy](../05-architecture/data-contracts.md) — add nullable columns, never change existing ones

This is intentional conservatism. Spectral features at 1/min sampling are experimental. The time-domain features (mean, std, slope) and cross-sensor features (pressure ratio, temperature spreads) are more reliable for anomaly detection at this sampling rate and should be the primary feature set.

### Validation Protocol for Spectral Features

Before adding spectral features to the production schema, run the following validation protocol to confirm they improve anomaly detection:

**Cohort selection**: select 1,000 devices with known anomaly events (from maintenance logs or operator feedback) and 1,000 control devices with no reported issues over the same period. Both cohorts should span multiple equipment models and geographic regions to avoid confounding.

**A/B feature set comparison**:
- Feature set A (baseline): time-domain + cross-sensor features only (current Contract 3 columns)
- Feature set B (candidate): baseline features + spectral features (spectral entropy, dominant frequency, spectral energy ratio)

Train the same anomaly detection model (Isolation Forest with identical hyperparameters) on both feature sets using the same temporal train/test split.

**Primary metric**: precision@k, where k = the top 5% of anomaly scores per fleet cohort. This measures whether the highest-scored devices are truly anomalous. Precision@k is preferred over AUC for unsupervised anomaly detection because it evaluates the actionable tail of the score distribution.

**Minimum improvement threshold**: spectral features must improve precision@k by at least 5% (absolute) over the baseline feature set. Marginal improvements below this threshold do not justify the added compute cost and schema complexity.

**Generalization check**: validate on a held-out 3-month period not used during development. If spectral features improve precision during the development period but not during the held-out period, they are overfitting to temporal artifacts and should not be promoted to production.

Document results in an [experiment tracking](../06-modeling/experiment-tracking.md) entry. The decision to add spectral columns to Contract 3 requires agreement between the Feature Engineer and ML Scientist.

## What Would Require Higher-Rate Data

For completeness, here's what frequency analysis could do with proper sampling rates — and what sensor infrastructure changes would be needed:

| Analysis | Required Sampling Rate | What It Detects | Sensor Change Needed |
|----------|----------------------|-----------------|---------------------|
| Bearing defect detection | 1–10 kHz | Inner/outer race defects, ball pass frequencies | Dedicated accelerometer with high-speed DAQ |
| Motor current signature | 1–5 kHz | Broken rotor bars, eccentricity, electrical faults | Current transformer with high-speed sampling |
| Refrigerant flow noise | 500 Hz–5 kHz | Thermostatic expansion valve (TXV) hunting, flash gas | Acoustic sensor near TXV |
| Compressor valve analysis | 5–20 kHz | Valve leakage, plate wear | High-frequency pressure transducer |

These are outside the current [project scope](../05-architecture/system-overview.md) but represent a natural extension if the business case for predictive maintenance justifies the sensor investment. The feature engineering framework in Foundry would need a separate high-frequency ingestion path but could reuse the same windowing and [feature store](./feature-store.md) patterns.

## Summary

Frequency-domain features at 1/min sampling are a narrow but real capability: they detect slow periodicities like compressor short-cycling and defrost regularity. They do not replace proper vibration analysis. Treat spectral features as supplementary to the [time-domain](./time-domain.md) and [cross-sensor](./cross-sensor.md) features, validate them empirically before adding to the production schema, and be transparent with stakeholders about what they can and cannot detect.

## Related Documents

- [Time-Domain Features](./time-domain.md) — primary feature set (mean, std, min, max, slope, p95)
- [Cross-Sensor Features](./cross-sensor.md) — derived features from sensor combinations
- [Windowing Strategies](./windowing.md) — window sizes and alignment that determine FFT sample counts
- [Feature Store](./feature-store.md) — how features are stored and served, including schema evolution for new features
- [Data Contracts — Contract 3](../05-architecture/data-contracts.md) — current production schema (does not yet include spectral features)
- [Transform Patterns](../04-palantir/transform-patterns.md) — Pandas UDF patterns for custom computations in Foundry
