# UAV Telemetry Dataset — From ~31 GB of ULog Data to a Curated CSV

## Overview

This document explains how the project transformed the original PX4 UAV-SEAD telemetry collection — approximately **31 GB and ~3,196 PX4 ULog flight files** — into the curated machine-learning dataset stored in this repository.

The final CSV is **not a random sample of the original data**. It is a deliberately constructed analytical dataset produced through:

```text
~31 GB / ~3,196 PX4 ULogs
        |
        v
Telemetry profiling
        |
        v
Required-topic filtering
        |
        v
Raw-column selection
        |
        v
Complete-flight filtering
        |
        v
ULog extraction
        |
        v
Multi-rate synchronization
        |
        v
Common 10 Hz timeline
        |
        v
10-second window assignment
        |
        v
One wide CSV
        |
        v
Missing-value handling
        |
        v
Constant-feature analysis
        |
        v
EDA
        |
        v
ML feature engineering
```

---

# 1. Original Dataset

The starting dataset consists of PX4 `.ulg` flight logs.

Approximate scale:

| Property | Original dataset |
|---|---:|
| Storage | ~31 GB |
| ULog files | ~3,196 |
| Format | PX4 ULog |
| Telemetry | Multiple PX4 topics |
| Sampling | Topic-dependent |

A ULog is not a single rectangular table. A flight contains many independent telemetry streams, and each stream can have a different logging frequency, duration, and set of fields.

Examples include:

```text
battery_status
vehicle_gps_position
vehicle_local_position
vehicle_attitude
sensor_combined
sensor_baro
estimator_status
ekf2_innovations
actuator_controls_0
actuator_outputs
...
```

Therefore, the original 31 GB cannot be treated as one normal CSV.

---

# 2. Why the Final Dataset Is Much Smaller

The reduction from approximately:

```text
31 GB
~3,196 ULog files
```

to approximately:

```text
6,200–6,400 synchronized rows
54 columns
```

happens through **multiple deliberate transformations**.

It is not one compression operation.

The main reductions are:

1. **Flight selection**
2. **Topic selection**
3. **Raw-column selection**
4. **Multi-rate synchronization**
5. **10 Hz common timeline**
6. **10-second window organization**
7. **Missing-value handling**
8. **Constant/uninformative feature removal**

The original ULogs remain the source data. The CSV is a derived analytical dataset.

> The exact row count should always be taken from the current CSV in `data/`, because the dataset may be regenerated or extended.

---

# 3. Step 1 — Understand the PX4 Telemetry

Before extracting data, the project first analyzed what each PX4 topic represents.

The goal was to cover the major UAV subsystems:

```text
Power
Navigation
Position estimation
Attitude
IMU
Altitude
State estimation
EKF innovations
Flight control
Actuation
Communication
Pilot input
System health
```

This engineering analysis was used to decide which topics and fields were useful for anomaly detection.

---

# 4. Step 2 — Select Required Topics

The project does not use every topic contained in a PX4 ULog.

The selected telemetry topics are:

```text
battery_status
system_power
vehicle_gps_position
vehicle_local_position
vehicle_attitude
vehicle_attitude_setpoint
sensor_combined
sensor_baro
estimator_status
ekf2_innovations
actuator_controls_0
actuator_outputs
telemetry_status
input_rc
cpuload
vehicle_status_flags
```

These topics collectively describe the aircraft's:

- Power condition
- GPS/navigation state
- Local position
- Attitude and angular motion
- IMU behavior
- Barometric state
- EKF health
- Sensor-estimator disagreement
- Controller commands
- Actuator outputs
- Communication state
- Pilot input
- Computational health

---

# 5. Step 3 — Filter Flights

Not every ULog contains every required topic.

Therefore, each flight was checked for telemetry availability.

Conceptually:

```text
ULog
 |
 +-- Required topics present?
       |
       +-- NO  -> do not use for complete-data dataset
       |
       +-- YES -> continue
```

During profiling, **37 flights** were identified as containing the required topic set.

A stricter check was then applied:

> Does the flight also contain every raw column selected for the final dataset?

This further reduced the usable set.

This is intentional. The project prefers a smaller set of flights with consistent telemetry rather than mixing incompatible schemas.

The current curated CSV contains the subset that passed the final complete-data criteria.

---

# 6. Step 4 — Select Raw Columns

Even when a required topic exists, we do not keep every field inside it.

For example, `vehicle_gps_position` contains many fields such as:

```text
lat
lon
alt
eph
epv
hdop
vdop
vel_m_s
satellites_used
jamming_indicator
...
```

Only fields relevant to the current anomaly-detection architecture are retained.

The selection is based on:

- Engineering meaning
- Anomaly relevance
- Redundancy
- Variation across flights
- Interpretability
- Computational cost

This prevents the final dataset from becoming unnecessarily high-dimensional.

---

# 7. Step 5 — Extract the Selected Raw Telemetry

The ULog extraction script reads each selected ULog and extracts only:

```text
required topics
+
selected raw columns
```

The extracted data is still a time series.

For example:

```text
vehicle_gps_position
timestamp | eph | epv | vel_m_s | satellites_used | ...
```

and:

```text
sensor_combined
timestamp | gyro_0 | gyro_1 | gyro_2 | accel_0 | accel_1 | accel_2
```

At this point, these are still separate telemetry streams.

---

# 8. Step 6 — Why We Cannot Simply Join the Topics

Different PX4 topics have different logging rates.

For example:

```text
IMU                 -> high frequency
Attitude            -> high frequency
Actuator outputs    -> high frequency
GPS                 -> lower frequency
Battery             -> lower frequency
EKF status          -> lower frequency
CPU load             -> lower frequency
```

A single flight can therefore contain:

```text
sensor_combined          16,000+ records
vehicle_gps_position        hundreds
battery_status              hundreds
cpuload                     tens
```

If these were joined using row number, the values would not refer to the same moment in the flight.

We therefore use **timestamp-based synchronization**.

---

# 9. Step 7 — Common 10 Hz Timeline

All selected topics are brought onto a common:

```text
10 Hz
```

timeline.

10 Hz means:

```text
10 samples per second
```

or approximately:

```text
1 sample every 0.1 seconds
```

The common timeline therefore looks like:

```text
00:00:00.000
00:00:00.100
00:00:00.200
00:00:00.300
00:00:00.400
...
```

Each selected topic is aligned to this timeline.

This creates a common temporal reference for the machine-learning pipeline.

---

# 10. Why 10 Hz?

The project combines both high-frequency and low-frequency telemetry.

A common rate is required so that:

```text
GPS
+
IMU
+
attitude
+
EKF
+
battery
+
actuator
+
system health
```

can appear in the same row.

10 Hz was selected as a practical common analysis rate that preserves useful flight behavior while avoiding the enormous row count produced by keeping every high-frequency raw sample.

The original ULog files are not modified by this process.

---

# 11. Step 8 — Merge Topics into One Wide CSV

After synchronization, separate telemetry streams are combined horizontally.

Conceptually:

```text
                  Common 10 Hz timeline
                           |
        +------------------+------------------+
        |                  |                  |
       GPS                IMU              Battery
        |                  |                  |
        +------------------+------------------+
                           |
                           v
                  One synchronized row
```

One row therefore contains telemetry from multiple subsystems at approximately the same flight instant.

The column naming convention is:

```text
topic__field
```

For example:

```text
vehicle_gps_position__vel_m_s
```

means:

```text
Topic = vehicle_gps_position
Field = vel_m_s
```

---

# 12. Step 9 — 10-Second Windows

The synchronized 10 Hz data is organized into 10-second windows.

At 10 Hz:

```text
10 seconds × 10 samples/second
≈ 100 samples/window
```

Conceptually:

```text
Flight
 |
 +-- Window 0
 |     ~100 samples
 |
 +-- Window 1
 |     ~100 samples
 |
 +-- Window 2
 |     ~100 samples
 |
 +-- ...
```

The CSV still contains the individual 10 Hz rows.

We do **not** collapse an entire flight into one row.

We also do **not** initially collapse every 10-second window into one row.

The raw synchronized samples are retained so that temporal feature engineering can be performed later.

---

# 13. Why Keep Raw Rows?

An anomaly is often a temporal event.

For example:

```text
Normal
Normal
Normal
GPS degradation
GPS degradation
GPS degradation
Normal
```

If the entire flight were represented by one row, we would lose:

- When the event happened
- How long it lasted
- What changed before it
- What changed during it
- What changed afterward

The current CSV therefore preserves the temporal structure.

Later, approximately 100 samples belonging to one 10-second window will be transformed into a window-level feature vector.

---

# 14. Metadata Columns

The dataset contains metadata used to reconstruct the temporal structure.

### `flight_id`

Identifies the source flight.

It is essential for:

- Flight-wise grouping
- Plotting
- Train/test splitting
- Preventing data leakage

### `window_id`

Identifies the 10-second window within a flight.

The combination:

```text
flight_id + window_id
```

identifies a particular flight segment.

### `window_start`

Start time of the window.

### `window_end`

End time of the window.

### `time`

Human-readable flight-relative time:

```text
HH:MM:SS:MS
```

### `elapsed_seconds`

Numeric flight-relative time used for calculations and plotting.

---

# 15. Step 10 — Missing Values

Because different topics have different original rates, synchronization can create missing values.

For example:

```text
GPS updates less frequently than IMU
```

so there may not be a new GPS measurement at every 100 ms point.

The cleaned dataset therefore uses a separate missing-value handling stage:

```text
Synchronized dataset
        |
        v
Missing-value report
        |
        v
Forward fill
        |
        v
Backward fill
        |
        v
Verification
        |
        v
Clean dataset
```

The cleaning step is kept separate from the ULog extraction step so that the original extraction remains reproducible.

---

# 16. Step 11 — Constant Feature Analysis

After cleaning, the dataset was checked for features that do not change.

For example:

```text
1
1
1
1
1
```

A constant feature provides no useful variation for an anomaly detector.

Such features were identified and removed from the final analytical feature set.

This is one reason the number of columns changed between early raw extraction reports and the final curated CSV.

---

# 17. Current Dataset Schema

The current curated dataset contains metadata plus selected telemetry.

The major telemetry topics are:

```text
battery_status
system_power
vehicle_gps_position
vehicle_local_position
vehicle_attitude
vehicle_attitude_setpoint
sensor_combined
sensor_baro
estimator_status
ekf2_innovations
actuator_controls_0
actuator_outputs
cpuload
```

Additional telemetry topics were evaluated during development, while the final feature set was reduced through the feature-selection and constant-feature analysis stages.

The current file should be treated as the source of truth for the exact final column list and row count.

---

# 18. Why the Final CSV Is Only ~6,200–6,400 Rows

The approximate row count can be understood mathematically.

At 10 Hz:

```text
1 second  = 10 rows
10 seconds = ~100 rows
70 seconds = ~700 rows
```

Therefore, if roughly ten usable flights contribute around 60–70 seconds each:

```text
~10 flights × ~650–700 rows
≈ ~6,500–7,000 rows
```

The current curated file is in this range.

The exact number can differ because flights have different durations and because only valid synchronized flight intervals are retained.

---

# 19. Why the 31 GB Does Not Need to Become 31 GB of CSV

The original 31 GB contains much more information than our current research question needs.

It includes:

```text
many flights
many topics
many fields
high-frequency signals
diagnostic information
redundant signals
signals not selected for this study
```

The project intentionally constructs a smaller representation.

The purpose is:

> **Preserve the telemetry information needed to study UAV anomalies, not preserve every byte of the original logging system.**

This is similar to building an analytical dataset from a much larger raw data warehouse.

---

# 20. What Is Lost?

The reduction is not lossless.

We intentionally do not retain:

- Every original PX4 topic
- Every original field
- Every high-frequency sample
- Every ULog file
- All raw diagnostic data

This is acceptable because the original ULog collection remains the source dataset.

The curated CSV is an **analysis layer**.

If a later experiment requires another PX4 field, the extraction pipeline can be rerun against the original ULogs.

---

# 21. What Is Preserved?

The curated dataset preserves the major information required by the current anomaly-detection architecture:

```text
Power
Navigation
Position
Attitude
IMU
Altitude
Estimator health
EKF innovations
Control
Actuation
System health
```

It also preserves:

```text
flight identity
window identity
time
elapsed time
```

Therefore, temporal and subsystem-level relationships can still be studied.

---

# 22. No Manual Anomaly Label in the Final CSV

The current CSV intentionally does not contain:

```text
anomaly = 0
anomaly = 1
```

The first ML approach is unsupervised.

The objective is to learn the structure of flight telemetry and identify unusual windows rather than forcing the model to reproduce labels assigned beforehand.

This follows the project's requirement to investigate whether the telemetry itself contains detectable anomaly patterns.

---

# 23. Next Stage: Window-Level Feature Engineering

The current CSV is a **raw synchronized dataset**, not yet the final machine-learning feature matrix.

Currently:

```text
1 row = 100 ms telemetry sample
```

The next stage will group rows using:

```text
flight_id + window_id
```

so that:

```text
~100 rows
      ↓
one 10-second window
```

For continuous signals we can calculate:

```text
mean
median
standard deviation
minimum
maximum
range
RMS
first value
last value
delta
linear trend
```

Example:

```text
vehicle_gps_position__vel_m_s
```

can become:

```text
gps_vel_mean
gps_vel_std
gps_vel_min
gps_vel_max
gps_vel_range
gps_vel_rms
gps_vel_delta
gps_vel_trend
```

---

# 24. Domain-Specific Features

Generic statistics will be combined with UAV-specific features.

Examples:

### GPS

```text
GPS velocity variability
GPS uncertainty
satellite-count variation
jamming-indicator behavior
```

### IMU

```text
gyro RMS
accelerometer RMS
axis variability
vibration-related features
```

### EKF

```text
innovation magnitude
innovation variability
test-ratio behavior
persistent innovation deviations
```

### Battery

```text
voltage drop
current variability
remaining-capacity trend
```

### Actuators

```text
output variability
motor/output imbalance
control effort
saturation-related behavior
```

---

# 25. Planned Machine-Learning Pipeline

After feature engineering:

```text
Curated Raw CSV
       |
       v
10-second grouping
       |
       v
Window-level features
       |
       v
Feature validation
       |
       v
Scaling where appropriate
       |
       v
Isolation Forest
       |
       v
Anomaly score
       |
       v
Threshold selection
       |
       v
Temporal anomaly events
       |
       v
Telemetry interpretation
       |
       v
Mission risk assessment
```

---

# 26. Why Isolation Forest?

The first planned anomaly-detection baseline is Isolation Forest because the current dataset does not contain a trusted anomaly label for every window.

The model can identify observations that are unusual relative to the learned feature distribution.

Later, other approaches may be investigated:

```text
Isolation Forest
One-Class SVM
Local Outlier Factor
Autoencoder
Temporal models
```

The more complex methods will only be useful if they provide meaningful improvement.

---

# 27. Avoiding Data Leakage

The dataset is time-series data, so random row-level train/test splitting is inappropriate.

For example, this should be avoided:

```text
Same flight
├── training rows
└── testing rows
```

because adjacent windows from the same flight are highly correlated.

Future evaluation should preferably be flight-wise:

```text
Training flights
       |
       v
Model learns behavior

Unseen flight
       |
       v
Anomaly detection
```

This provides a more realistic test of generalization.

---

# 28. Future Anomaly Interpretation

The final system should not simply output:

```text
ANOMALY
```

It should help answer:

```text
Why was this window unusual?
Which subsystem changed?
Which telemetry features contributed?
Did the behavior persist?
```

For example:

```text
Anomalous Window
      |
      +-- GPS uncertainty increased
      +-- satellite count changed
      +-- EKF innovation increased
      +-- local-position behavior changed
      +-- actuator response changed
```

This is the bridge from simple anomaly detection to **mission risk assessment**.

---

# 29. Recommended Repository Structure

```text
Major_Project/
|
├── README.md
|
├── docs/
|   └── DATASET_README.md
|
├── data/
|   └── flight_merged_final.csv
|
├── preprocessing/
|   ├── ulogToCSV.py
|   ├── clean_dataset.py
|   └── ...
|
├── notebooks/
|   ├── 01_Flight_Exploration.ipynb
|   ├── 02_EDA.ipynb
|   └── ...
|
└── models/
    └── ...
```

The original 31 GB ULog collection should normally remain outside GitHub. The repository should contain the processing code and the curated dataset needed to reproduce and understand the analysis.

---

# 30. Current Status

### Completed

- [x] Obtained original PX4 ULog dataset
- [x] Profiled thousands of ULog files
- [x] Studied PX4 telemetry topics
- [x] Selected relevant telemetry topics
- [x] Selected relevant raw columns
- [x] Filtered flights for required telemetry
- [x] Extracted selected ULog data
- [x] Resolved different logging rates
- [x] Created common 10 Hz timeline
- [x] Added 10-second window architecture
- [x] Merged topics into one wide CSV
- [x] Generated missing-value reports
- [x] Filled missing synchronized values
- [x] Performed constant-feature analysis
- [x] Performed EDA
- [x] Created the curated raw telemetry dataset

### Next

- [ ] Final dataset validation
- [ ] Window-level feature engineering
- [ ] Statistical feature generation
- [ ] Domain-specific feature generation
- [ ] Feature scaling
- [ ] Isolation Forest baseline
- [ ] Anomaly-score analysis
- [ ] Temporal anomaly-event detection
- [ ] Flight-wise evaluation
- [ ] Anomaly visualization
- [ ] Subsystem-level interpretation
- [ ] Mission risk assessment
- [ ] Model comparison

---

# 31. One-Sentence Explanation

> **We did not randomly reduce 31 GB of UAV data to a few thousand rows; we converted heterogeneous PX4 ULog streams into a deliberately selected, synchronized 10 Hz time-series dataset containing the telemetry required for UAV anomaly detection while preserving flight and 10-second-window structure for downstream temporal analysis.**
