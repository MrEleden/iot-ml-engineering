# Schema Management & Evolution

## The Problem

IoT sensor fleets are heterogeneous. Devices run different firmware versions, new sensor types get added, fields get renamed. If your pipeline assumes a fixed schema, it will break.

Real-world examples:
- Firmware v2.3 adds a `humidity` sensor to units that previously only reported `temperature` and `pressure`
- A field is renamed from `temp` to `temperature` in firmware v3.0
- A new `vibration_rms` field appears as accelerometers are retrofitted to older units
- Units in different regions report in different measurement units (Celsius vs Fahrenheit)

## Schema-on-Read vs Schema-on-Write

### Schema-on-Write (recommended for landing zone)

Enforce a minimal common schema at write time. Reject or quarantine messages that don't conform.

```python
from pyspark.sql.types import StructType, StructField, StringType, DoubleType, TimestampType

LANDING_SCHEMA = StructType([
    StructField("device_id", StringType(), nullable=False),
    StructField("timestamp", TimestampType(), nullable=False),
    StructField("sensor_type", StringType(), nullable=False),
    StructField("value", DoubleType(), nullable=False),
    StructField("unit", StringType(), nullable=False),
])
```

**Why**: catching bad data at ingestion is 10x cheaper than debugging a model that silently trained on corrupted features.

### Schema-on-Read (for metadata / semi-structured fields)

For the `metadata` blob, use schema-on-read. Store it as a JSON string or a MapType, and parse what you need downstream.

```python
# Landing: store metadata as-is
df_raw = df.withColumn("metadata", col("metadata").cast("string"))

# Feature engineering: extract what you need
df_features = (
    df_raw
    .withColumn("site_id", get_json_object(col("metadata"), "$.site_id"))
    .withColumn("firmware", get_json_object(col("metadata"), "$.firmware"))
)
```

## Delta Lake Schema Evolution

Delta Lake supports two modes of schema evolution:

### Additive (safe — new columns)

```python
(
    df_new_firmware
    .write
    .format("delta")
    .option("mergeSchema", "true")  # allows new columns
    .mode("append")
    .save("s3://datalake/raw/sensor_telemetry/")
)
```

New columns appear as `NULL` for historical rows. This is fine — your feature engineering code should handle nulls for optional sensors.

### Breaking (dangerous — type changes, renames)

Never allow implicit type changes. Instead:

1. **Version your topics** — `sensor-telemetry-v2` for the new schema
2. **Run both in parallel** — old devices write to v1, new devices to v2
3. **Merge in the batch layer** — a PySpark job reads both, normalizes, and writes to a unified table

```python
# Normalize v1 and v2 into a common schema
df_v1 = spark.read.format("delta").load(".../v1/").withColumn("humidity", lit(None))
df_v2 = spark.read.format("delta").load(".../v2/")

df_unified = df_v1.unionByName(df_v2, allowMissingColumns=True)
```

## Unit Normalization

Handle this at the earliest opportunity (landing or first transform):

```python
UNIT_CONVERSIONS = {
    ("fahrenheit", "celsius"): lambda v: (v - 32) * 5.0 / 9.0,
    ("psi", "bar"): lambda v: v * 0.0689476,
    ("inHg", "hPa"): lambda v: v * 33.8639,
}

@udf(DoubleType())
def normalize_unit(value, from_unit, to_unit):
    if from_unit == to_unit:
        return value
    key = (from_unit.lower(), to_unit.lower())
    if key in UNIT_CONVERSIONS:
        return UNIT_CONVERSIONS[key](value)
    return None  # quarantine unknown conversions
```

## Quarantine Pattern

Messages that fail schema validation go to a quarantine table, not the void:

```
landing/
├── sensor_telemetry/       ← valid messages
└── sensor_telemetry_quarantine/  ← invalid messages + error reason
```

```python
valid, invalid = validate_schema(df_raw)
valid.write.format("delta").mode("append").save(".../sensor_telemetry/")
invalid.write.format("delta").mode("append").save(".../sensor_telemetry_quarantine/")
```

Review the quarantine table weekly. Common patterns (new firmware, new sensor type) inform schema evolution decisions.
