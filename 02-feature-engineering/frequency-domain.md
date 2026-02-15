# Frequency-Domain Features

Sensor data often contains periodic patterns (compressor cycles, day/night temperature swings, vibration harmonics) that are invisible in time-domain statistics but obvious in the frequency domain.

## When to Use Frequency Features

- **Vibration sensors** — bearing faults show up as specific frequency peaks
- **Temperature/pressure** — compressor cycling creates periodic patterns; cycle frequency changes indicate degradation
- **Current/voltage** — electrical faults produce harmonic distortions

## FFT in PySpark

PySpark doesn't have a native FFT. Two approaches:

### Approach 1: Pandas UDF (recommended for moderate scale)

```python
import numpy as np
from pyspark.sql.functions import pandas_udf
from pyspark.sql.types import StructType, StructField, DoubleType, ArrayType

@pandas_udf(ArrayType(DoubleType()))
def compute_fft_magnitudes(values: pd.Series) -> pd.Series:
    """Compute FFT magnitude spectrum for each window of sensor values."""
    results = []
    for v in values:
        if v is None or len(v) < 8:
            results.append(None)
            continue
        arr = np.array(v)
        # Remove DC component, apply Hanning window to reduce spectral leakage
        arr = arr - np.mean(arr)
        arr = arr * np.hanning(len(arr))
        fft_vals = np.abs(np.fft.rfft(arr))
        # Normalize by window length
        fft_vals = fft_vals / len(arr)
        results.append(fft_vals.tolist())
    return pd.Series(results)
```

### Approach 2: Pre-aggregate in PySpark, FFT in post-processing

For very large datasets, collect windowed arrays in PySpark, then compute FFT in a separate step:

```python
from pyspark.sql import functions as F

# Collect values into arrays per device per window
df_windowed = (
    df
    .withColumn("window_id", F.window("timestamp", "1 hour"))
    .groupBy("device_id", "window_id")
    .agg(
        F.sort_array(F.collect_list(F.struct("timestamp", "value"))).alias("readings")
    )
    .withColumn("values", F.transform(F.col("readings"), lambda x: x.value))
)

# Apply FFT via Pandas UDF
df_fft = df_windowed.withColumn("fft_magnitudes", compute_fft_magnitudes(F.col("values")))
```

## Derived Frequency Features

Raw FFT output is high-dimensional. Extract meaningful scalar features:

```python
@pandas_udf(DoubleType())
def spectral_entropy(fft_magnitudes: pd.Series) -> pd.Series:
    """Entropy of the normalized power spectrum. High = broadband noise. Low = dominant frequency."""
    results = []
    for mags in fft_magnitudes:
        if mags is None or len(mags) == 0:
            results.append(None)
            continue
        power = np.array(mags) ** 2
        power_norm = power / (np.sum(power) + 1e-10)
        entropy = -np.sum(power_norm * np.log2(power_norm + 1e-10))
        results.append(float(entropy))
    return pd.Series(results)

@pandas_udf(DoubleType())
def dominant_frequency(fft_magnitudes: pd.Series) -> pd.Series:
    """Index of the strongest frequency component (excluding DC)."""
    results = []
    for mags in fft_magnitudes:
        if mags is None or len(mags) < 2:
            results.append(None)
            continue
        # Skip index 0 (DC component)
        dominant_idx = int(np.argmax(mags[1:])) + 1
        results.append(float(dominant_idx))
    return pd.Series(results)

@pandas_udf(DoubleType())
def spectral_centroid(fft_magnitudes: pd.Series) -> pd.Series:
    """Weighted average frequency — shifts as equipment degrades."""
    results = []
    for mags in fft_magnitudes:
        if mags is None or len(mags) == 0:
            results.append(None)
            continue
        freqs = np.arange(len(mags))
        mags_arr = np.array(mags)
        centroid = np.sum(freqs * mags_arr) / (np.sum(mags_arr) + 1e-10)
        results.append(float(centroid))
    return pd.Series(results)
```

### Feature interpretation for predictive maintenance

| Feature | Low value | High value | Maintenance signal |
|---------|-----------|------------|-------------------|
| Spectral entropy | Dominant single frequency (normal cycling) | Broadband noise (mechanical wear) | High entropy → inspect |
| Dominant frequency | Low-frequency cycles (normal) | Shifted from baseline | Frequency shift → bearing/motor issue |
| Spectral centroid | Energy in low frequencies | Energy shifting to high frequencies | Upward drift → increasing friction/wear |

## Sampling Rate Considerations

FFT requires evenly-spaced samples. IoT sensors rarely provide this.

**Solution**: resample before FFT:

```python
@pandas_udf(ArrayType(DoubleType()))
def resample_and_fft(timestamps: pd.Series, values: pd.Series) -> pd.Series:
    """Resample irregular sensor data to uniform spacing, then FFT."""
    results = []
    for ts_arr, val_arr in zip(timestamps, values):
        if ts_arr is None or len(ts_arr) < 8:
            results.append(None)
            continue
        # Create a pandas Series with datetime index
        s = pd.Series(val_arr, index=pd.to_datetime(ts_arr))
        # Resample to 1-second intervals, forward-fill gaps up to 5 seconds
        s_uniform = s.resample("1s").mean().ffill(limit=5)
        s_uniform = s_uniform.dropna()
        if len(s_uniform) < 8:
            results.append(None)
            continue
        arr = s_uniform.values - np.mean(s_uniform.values)
        fft_vals = np.abs(np.fft.rfft(arr)) / len(arr)
        results.append(fft_vals.tolist())
    return pd.Series(results)
```

**Key rule**: document your resampling frequency. A feature computed at 1Hz vs 10Hz will have completely different frequency bins, and anyone who reuses the feature without knowing this will get wrong results.
