# PX4 UAV Flight Telemetry Dataset

## 1. Dataset Overview

This dataset is the consolidated raw telemetry dataset prepared for the
**UAV Flight Anomaly Detection** project.

The source data comes from **PX4 ULog flight logs**. The ULog files were
processed, filtered to the selected telemetry topics and raw columns,
synchronized to a common **10 Hz** timeline, divided into **10-second
windows**, and merged into one CSV/XLSX dataset.

### Current dataset statistics

  Property                                     Value
  -------------------------------- -----------------
  Rows                                     **6,201**
  Columns                                     **54**
  Flight logs represented                      **9**
  Telemetry topics                            **13**
  Metadata columns                             **6**
  Raw telemetry columns                       **48**
  Common sampling rate                     **10 Hz**
  Time interval between rows         **0.1 seconds**
  Window duration                     **10 seconds**
  Missing values in current file               **0**
  Manual anomaly labels                     **None**

> **Important:** Earlier stages of the project considered a larger pool
> of flights. This README describes the actual contents of the attached
> `flight_merged_final` dataset: it currently contains **9 flight IDs
> and 6,201 rows**. The dataset should therefore be treated as the
> source of truth for the current experiment.

------------------------------------------------------------------------

# 2. What Does One Row Represent?

A row in this dataset represents **one synchronized telemetry sample at
10 Hz**.

Therefore:

-   10 rows ≈ 1 second
-   100 rows ≈ 10 seconds
-   Each row represents approximately **100 ms** of flight time.
-   The row contains telemetry values from multiple PX4 subsystems
    aligned to the same common timestamp.

The dataset is **not** organized as one row per flight and **not**
organized as one row per 10-second window.

Instead, the structure is:

``` text
Flight
  |
  +-- Window 0 (10 seconds)
  |      |
  |      +-- sample 0   00:00:00.000
  |      +-- sample 1   00:00:00.100
  |      +-- sample 2   00:00:00.200
  |      +-- ...
  |      +-- sample 99  00:00:09.900
  |
  +-- Window 1 (10 seconds)
  |      |
  |      +-- sample 100
  |      +-- ...
  |
  +-- Window 2
         ...
```

This structure is intentional because anomaly detection needs to
preserve the **temporal behavior** of the aircraft rather than reducing
an entire flight to a single row.

------------------------------------------------------------------------

# 3. What Is `flight_id`?

`flight_id` identifies the source flight/log from which a row
originated.

Examples in the current dataset include:

``` text
18_57_00
18_58_45
19_12_45
19_16_20
19_31_44
19_38_12
19_39_24
19_34_00
19_35_07
```

The same `flight_id` appears on every row belonging to that flight.

This is extremely important for later machine-learning work because the
model should be evaluated **flight-wise**, not by randomly mixing rows
from the same flight between training and testing.

------------------------------------------------------------------------

# 4. What Is `window_id`?

`window_id` identifies the 10-second temporal segment to which a row
belongs.

For example:

``` text
window_id = 0
window_start = 00:00:00:000
window_end   = 00:00:10:000
```

The next window is:

``` text
window_id = 1
window_start = 00:00:10:000
window_end   = 00:00:20:000
```

and so on.

The `window_id` is used together with `flight_id`. A window number by
itself should not be interpreted as globally unique because window
numbering can restart for a new flight.

The combination:

``` text
flight_id + window_id
```

identifies a particular flight segment.

------------------------------------------------------------------------

# 5. Why Use 10-Second Windows?

A 10-second window provides enough temporal context to observe changes
in aircraft behavior while keeping the data manageable.

A very short window may contain only a few sensor changes and may not
capture the development of an abnormal event.

A very long window can mix several different flight states together and
may make it harder for an anomaly detector to localize the event.

The chosen architecture therefore preserves both:

1.  **Raw time-series information** at 10 Hz.
2.  **Temporal context** through 10-second windows.

The current CSV keeps every 10 Hz sample rather than collapsing each
window into a single row. Feature engineering will be performed later.

------------------------------------------------------------------------

# 6. What Is `time`?

`time` is the human-readable timestamp within the flight timeline.

It is stored in:

``` text
HH:MM:SS:MS
```

Example:

``` text
00:00:00:000
00:00:00:100
00:00:00:200
00:00:00:300
```

At the selected 10 Hz rate, the expected difference between consecutive
rows is approximately:

``` text
100 ms
```

------------------------------------------------------------------------

# 7. What Is `elapsed_seconds`?

`elapsed_seconds` is the numerical representation of the flight-relative
time.

Example:

``` text
0.0
0.1
0.2
0.3
...
```

This column is useful for numerical analysis, plotting, calculating
trends, and creating temporal features.

------------------------------------------------------------------------

# 8. Why Are There So Many Columns?

The aircraft does not have one sensor that describes whether a flight is
normal.

A flight is a complex system involving:

-   Power
-   GPS
-   Local position estimation
-   Attitude
-   Desired attitude
-   IMU sensors
-   Barometer
-   EKF state estimation
-   EKF innovations
-   Flight-control commands
-   Actuator outputs
-   CPU and memory health

Each subsystem produces different signals.

For example, an anomaly may appear as:

``` text
GPS degradation
        ↓
position uncertainty increases
        ↓
EKF innovation increases
        ↓
controller response changes
        ↓
actuator outputs change
```

If only GPS were considered, the downstream effects might be missed.

The dataset therefore combines complementary signals from several
subsystems.

The goal is **not** to keep every field available in PX4.

The project deliberately selected a smaller set of raw columns that are
relevant to flight behavior and anomaly detection.

------------------------------------------------------------------------

# 9. Column Naming Convention

Telemetry columns use the following naming convention:

``` text
topic__field
```

For example:

``` text
vehicle_gps_position__vel_m_s
```

means:

``` text
Topic:
vehicle_gps_position

Field:
vel_m_s
```

The double underscore (`__`) separates the PX4 topic name from the
selected raw field name.

This makes it possible to identify the origin of every feature without
requiring a separate schema file.

------------------------------------------------------------------------

# 10. Topic and Column Description

## 10.1 `battery_status`

### Columns

``` text
battery_status__voltage_v
battery_status__current_a
battery_status__remaining
```

### Purpose

Represents the electrical state of the battery during flight.

### Features

  -----------------------------------------------------------------------
  Column                              Meaning / Role
  ----------------------------------- -----------------------------------
  `voltage_v`                         Battery voltage. Sudden drops can
                                      indicate power-related events or
                                      high load.

  `current_a`                         Battery current. Useful for
                                      identifying changes in power
                                      demand.

  `remaining`                         Estimated remaining battery level.
                                      Useful for identifying abnormal
                                      battery depletion or unexpected
                                      changes.
  -----------------------------------------------------------------------

### Why it matters

A power problem can affect the aircraft's ability to maintain stable
flight and can also influence actuator behavior.

------------------------------------------------------------------------

# 11. `system_power`

### Column

``` text
system_power__voltage5V_v
```

### Purpose

Represents the aircraft's 5V power rail.

### Feature

  Column          Meaning / Role
  --------------- ---------------------------------
  `voltage5V_v`   Measured 5V power-rail voltage.

### Why it matters

Abnormal power-rail behavior can indicate electrical instability that
may affect sensors, peripherals, or flight-control hardware.

------------------------------------------------------------------------

# 12. `vehicle_gps_position`

GPS is one of the most important topics in this project.

### Columns

``` text
vehicle_gps_position__eph
vehicle_gps_position__epv
vehicle_gps_position__jamming_indicator
vehicle_gps_position__vel_m_s
vehicle_gps_position__satellites_used
```

### Purpose

Describes GPS quality and movement-related information.

  -----------------------------------------------------------------------
  Column                              Meaning / Role
  ----------------------------------- -----------------------------------
  `eph`                               Estimated horizontal position
                                      uncertainty.

  `epv`                               Estimated vertical position
                                      uncertainty.

  `jamming_indicator`                 Indicator related to GPS
                                      interference/jamming conditions.

  `vel_m_s`                           GPS-derived ground speed magnitude.

  `satellites_used`                   Number of satellites used by the
                                      GPS solution.
  -----------------------------------------------------------------------

### Why it matters

GPS degradation can be a major source of navigation anomalies.

The combination of satellite availability, position uncertainty, jamming
indication, and velocity provides more information than any one GPS
field alone.

------------------------------------------------------------------------

# 13. `vehicle_local_position`

### Columns

``` text
vehicle_local_position__x
vehicle_local_position__y
vehicle_local_position__z
vehicle_local_position__vz
vehicle_local_position__xy_valid
```

### Purpose

Represents the aircraft's local position and vertical motion.

  -----------------------------------------------------------------------
  Column                              Meaning / Role
  ----------------------------------- -----------------------------------
  `x`                                 Local X position.

  `y`                                 Local Y position.

  `z`                                 Local Z position.

  `vz`                                Vertical velocity.

  `xy_valid`                          Indicates whether the horizontal
                                      local-position estimate is valid.
  -----------------------------------------------------------------------

### Why it matters

Local-position behavior can reveal navigation instability, unexpected
movement, position jumps, or estimation problems.

------------------------------------------------------------------------

# 14. `vehicle_attitude`

### Columns

``` text
vehicle_attitude__rollspeed
vehicle_attitude__pitchspeed
vehicle_attitude__yawspeed
```

### Purpose

Represents rotational motion of the aircraft.

  Column         Meaning / Role
  -------------- ---------------------
  `rollspeed`    Roll angular rate.
  `pitchspeed`   Pitch angular rate.
  `yawspeed`     Yaw angular rate.

### Why it matters

Unexpected rotational motion can indicate instability, disturbances,
control problems, or abnormal flight behavior.

------------------------------------------------------------------------

# 15. `vehicle_attitude_setpoint`

### Columns

``` text
vehicle_attitude_setpoint__roll_body
vehicle_attitude_setpoint__pitch_body
vehicle_attitude_setpoint__thrust
```

### Purpose

Represents the desired attitude/control targets.

  Column         Meaning / Role
  -------------- -------------------------
  `roll_body`    Desired roll attitude.
  `pitch_body`   Desired pitch attitude.
  `thrust`       Desired thrust command.

### Why it matters

This topic allows the future model to compare:

``` text
What the controller requested
            vs
What the aircraft actually did
```

For example, a large difference between desired attitude and observed
response may indicate abnormal behavior.

------------------------------------------------------------------------

# 16. `sensor_combined`

### Columns

``` text
sensor_combined__gyro_rad_0
sensor_combined__gyro_rad_1
sensor_combined__gyro_rad_2

sensor_combined__accelerometer_m_s2_0
sensor_combined__accelerometer_m_s2_1
sensor_combined__accelerometer_m_s2_2
```

### Purpose

Contains selected IMU measurements.

  -----------------------------------------------------------------------
  Column group                        Purpose
  ----------------------------------- -----------------------------------
  `gyro_rad_0..2`                     Angular-rate measurements across
                                      the three axes.

  `accelerometer_m_s2_0..2`           Linear acceleration measurements
                                      across the three axes.
  -----------------------------------------------------------------------

### Why it matters

IMU signals are high-value indicators of:

-   Vibrations
-   Sudden acceleration
-   Rotational disturbances
-   Mechanical abnormalities
-   Unexpected flight dynamics

------------------------------------------------------------------------

# 17. `sensor_baro`

### Columns

``` text
sensor_baro__pressure
sensor_baro__altitude
```

### Purpose

Represents barometric measurements.

  Column       Meaning / Role
  ------------ -------------------------------------------------
  `pressure`   Atmospheric pressure measured by the barometer.
  `altitude`   Barometric altitude estimate.

### Why it matters

Barometric anomalies can indicate altitude-estimation problems or
unusual pressure measurements.

------------------------------------------------------------------------

# 18. `estimator_status`

### Columns

``` text
estimator_status__vel_test_ratio
estimator_status__pos_test_ratio
estimator_status__hgt_test_ratio
estimator_status__innovation_check_flags
```

### Purpose

Represents selected health and consistency information from the state
estimator.

  Column                     Meaning / Role
  -------------------------- ------------------------------------------
  `vel_test_ratio`           Velocity consistency/test ratio.
  `pos_test_ratio`           Position consistency/test ratio.
  `hgt_test_ratio`           Height consistency/test ratio.
  `innovation_check_flags`   Flags associated with innovation checks.

### Why it matters

This topic is particularly valuable because the estimator itself can
expose inconsistencies between sensor measurements and the estimated
aircraft state.

------------------------------------------------------------------------

# 19. `ekf2_innovations`

### Columns

``` text
ekf2_innovations__vel_pos_innov_0
ekf2_innovations__vel_pos_innov_1
ekf2_innovations__vel_pos_innov_2
ekf2_innovations__vel_pos_innov_3
ekf2_innovations__vel_pos_innov_4
ekf2_innovations__vel_pos_innov_5
```

### Purpose

Contains selected EKF innovation signals.

### Why it matters

Innovation represents the discrepancy between measurements and the
estimator's prediction/expected state.

This makes EKF innovation features especially useful for anomaly
detection.

A useful future analysis is:

``` text
normal innovation behavior
          ↓
increasing innovation magnitude
          ↓
persistent abnormal innovation
          ↓
potential anomaly
```

The six innovation components should be treated as separate raw signals
initially. Their exact semantic mapping should be documented from the
corresponding PX4 message/version if the thesis requires axis-by-axis
interpretation.

------------------------------------------------------------------------

# 20. `actuator_controls_0`

### Columns

``` text
actuator_controls_0__control_0
actuator_controls_0__control_1
actuator_controls_0__control_2
actuator_controls_0__control_3
```

### Purpose

Represents selected flight-controller control outputs before final
actuator output mapping.

### Why it matters

These values describe what the controller is requesting from the
actuators.

They are useful for understanding whether unusual aircraft behavior is
accompanied by unusual control commands.

------------------------------------------------------------------------

# 21. `actuator_outputs`

### Columns

``` text
actuator_outputs__output_0
actuator_outputs__output_1
actuator_outputs__output_2
actuator_outputs__output_3
```

### Purpose

Represents selected final actuator output channels.

### Why it matters

Actuator outputs provide information about how strongly the flight
controller is commanding the propulsion/control system.

Comparing:

``` text
actuator_controls_0
        vs
actuator_outputs
```

can help identify unusual control behavior or actuator-related patterns.

------------------------------------------------------------------------

# 22. `cpuload`

### Columns

``` text
cpuload__load
cpuload__ram_usage
```

### Purpose

Represents onboard computational resource usage.

  Column        Meaning / Role
  ------------- ----------------
  `load`        CPU load.
  `ram_usage`   RAM usage.

### Why it matters

Computational stress can potentially accompany abnormal system behavior,
timing issues, or degraded processing conditions.

These signals are therefore retained as system-health context.

------------------------------------------------------------------------

# 23. Complete Dataset Schema

The dataset contains six metadata columns:

``` text
flight_id
window_id
window_start
window_end
time
elapsed_seconds
```

and 48 selected telemetry columns:

``` text
battery_status
    voltage_v
    current_a
    remaining

system_power
    voltage5V_v

vehicle_gps_position
    eph
    epv
    jamming_indicator
    vel_m_s
    satellites_used

vehicle_local_position
    x
    y
    z
    vz
    xy_valid

vehicle_attitude
    rollspeed
    pitchspeed
    yawspeed

vehicle_attitude_setpoint
    roll_body
    pitch_body
    thrust

sensor_combined
    gyro_rad_0
    gyro_rad_1
    gyro_rad_2
    accelerometer_m_s2_0
    accelerometer_m_s2_1
    accelerometer_m_s2_2

sensor_baro
    pressure
    altitude

estimator_status
    vel_test_ratio
    pos_test_ratio
    hgt_test_ratio
    innovation_check_flags

ekf2_innovations
    vel_pos_innov_0
    vel_pos_innov_1
    vel_pos_innov_2
    vel_pos_innov_3
    vel_pos_innov_4
    vel_pos_innov_5

actuator_controls_0
    control_0
    control_1
    control_2
    control_3

actuator_outputs
    output_0
    output_1
    output_2
    output_3

cpuload
    load
    ram_usage
```

Therefore:

``` text
6 metadata columns
+
48 telemetry columns
=
54 total columns
```

------------------------------------------------------------------------

# 24. Why Were Other PX4 Fields Removed?

PX4 ULog files can contain a very large number of topics and fields.

Keeping everything would create several problems:

1.  Extremely high dimensionality.
2.  Many redundant variables.
3.  Constant or nearly constant variables.
4.  Signals unrelated to the anomaly types being investigated.
5.  Increased computational cost.
6.  Greater risk of overfitting.
7.  More difficult interpretation of detected anomalies.

The project therefore uses a **domain-driven feature selection
approach**.

The selected topics cover the major categories:

``` text
Power
GPS / Navigation
Position
Attitude
IMU
Altitude
State Estimation
EKF Innovations
Control
Actuation
System Health
```

This gives the model multiple independent views of aircraft behavior.

------------------------------------------------------------------------

# 25. Why Is There No Anomaly Label?

The current dataset intentionally contains **no manually assigned
anomaly label**.

There is no column such as:

``` text
anomaly = 0 / 1
```

This is deliberate.

The project is designed around **unsupervised anomaly detection**.

Instead of telling the model:

``` text
"This row is anomalous."
"This row is normal."
```

the objective is to allow the model to learn the structure of the
telemetry data and identify observations that are unusual relative to
the learned distribution.

This also avoids introducing subjective labels before the anomaly
patterns have been properly investigated.

------------------------------------------------------------------------

# 26. Important Point About the Current Raw Dataset

The current CSV should be considered the **raw synchronized analysis
dataset**, not the final machine-learning feature matrix.

The data is intentionally still at the sample level:

``` text
1 row = 100 ms telemetry sample
```

The next stage will transform these samples into meaningful temporal
features.

For example, within a 10-second window:

``` text
GPS speed
    mean
    standard deviation
    minimum
    maximum
    range
    RMS
    trend
    first-to-last change
```

The same concept can be applied to relevant continuous telemetry
signals.

This converts the raw time series into a compact representation of the
behavior of each flight window.

------------------------------------------------------------------------

# 27. Future Work

## Phase 1 --- Validate the Final Raw Dataset

Before machine learning, the dataset should be programmatically
validated.

Checks should include:

-   Number of flights.
-   Number of rows per flight.
-   Number of windows per flight.
-   10 Hz sampling consistency.
-   Duplicate rows.
-   Missing values.
-   Infinite values.
-   Unexpected data types.
-   Constant features.
-   Extreme numerical values.
-   Correct ordering by `flight_id`, `window_id`, and time.

The output should be a final dataset validation report.

------------------------------------------------------------------------

## Phase 2 --- Window-Level Feature Engineering

The raw 10 Hz samples will be grouped using:

``` text
flight_id + window_id
```

Each 10-second window will become one machine-learning observation.

For continuous telemetry features, candidate statistics include:

``` text
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
linear trend / slope
```

For example:

``` text
vehicle_gps_position__vel_m_s
```

could become:

``` text
gps_vel_mean
gps_vel_std
gps_vel_min
gps_vel_max
gps_vel_range
gps_vel_rms
gps_vel_delta
gps_vel_trend
```

This preserves the behavior of the signal without feeding every
individual 100 ms sample directly into the first anomaly detector.

------------------------------------------------------------------------

## Phase 3 --- Domain-Specific Features

In addition to generic statistics, domain-specific features should be
created.

### GPS

Potential features:

``` text
GPS speed variability
GPS position uncertainty trend
satellite-count changes
jamming-indicator changes
```

### Attitude / IMU

Potential features:

``` text
roll-rate variability
pitch-rate variability
yaw-rate variability
acceleration magnitude
gyro magnitude
high-frequency vibration indicators
```

### Estimator

Potential features:

``` text
velocity innovation magnitude
innovation variability
position test ratio statistics
height test ratio statistics
innovation flag changes
```

### Power

Potential features:

``` text
battery voltage drop
current variability
battery remaining change
5V rail variability
```

### Control / Actuation

Potential features:

``` text
control-output variability
actuator-output variability
control-to-output differences
```

These domain-specific features may be more informative than generic
statistics alone.

------------------------------------------------------------------------

# 28. Feature Scaling

After feature engineering, the feature distributions will be inspected.

Different signals have very different numerical scales:

``` text
battery voltage      ≈ tens
GPS speed            ≈ units
gyro                 ≈ fractions
EKF innovation       ≈ small values
CPU load             ≈ percentage-like values
```

Therefore, scaling will be evaluated before applying distance- or
distribution-sensitive algorithms.

Possible approaches include:

``` text
StandardScaler
RobustScaler
```

The final choice should be based on the observed distributions and the
presence of extreme values.

------------------------------------------------------------------------

# 29. Flight-Wise Train/Test Strategy

A critical point is **data leakage**.

Randomly splitting rows can be misleading because neighboring samples
from the same flight are highly correlated.

For example, this would be problematic:

``` text
Same flight
    ├── training rows
    └── testing rows
```

Instead, the evaluation should preferably be flight-wise:

``` text
Training flights
      ↓
Model learns normal flight behavior

Unseen flight
      ↓
Model produces anomaly scores
```

This gives a much more realistic test of whether the detector can
generalize to a previously unseen flight.

------------------------------------------------------------------------

# 30. Unsupervised Anomaly Detection

The first planned model is **Isolation Forest**.

The basic workflow will be:

``` text
Raw synchronized CSV
        ↓
10-second windows
        ↓
Statistical + domain features
        ↓
Feature validation
        ↓
Scaling
        ↓
Isolation Forest
        ↓
Anomaly score
        ↓
Anomaly threshold
        ↓
Normal / suspicious windows
```

Isolation Forest is appropriate as an initial baseline because it does
not require manually labelled anomaly examples.

The model will assign an anomaly score to each window.

------------------------------------------------------------------------

# 31. Choosing the Anomaly Threshold

The anomaly threshold should not simply be chosen arbitrarily.

Several approaches can be investigated:

### Contamination-based threshold

Specify an expected approximate proportion of anomalies.

### Score-distribution threshold

Analyze the distribution of anomaly scores and identify extreme
observations.

### Percentile-based threshold

For example, investigate the most extreme:

``` text
1%
2%
5%
```

of windows.

The final threshold should be justified experimentally rather than
selected only because it produces a convenient number of anomalies.

------------------------------------------------------------------------

# 32. Temporal Anomaly Confirmation

A single anomalous window does not necessarily mean that the aircraft
experienced a real anomaly.

Sensor noise can produce isolated abnormal observations.

Therefore, future work should investigate **temporal persistence**.

For example:

``` text
Window 10 → Normal
Window 11 → Anomalous
Window 12 → Anomalous
Window 13 → Anomalous
Window 14 → Normal
```

is more suspicious than:

``` text
Window 10 → Normal
Window 11 → Anomalous
Window 12 → Normal
Window 13 → Normal
```

A future anomaly-event layer can group consecutive anomalous windows
into a single event.

------------------------------------------------------------------------

# 33. Anomaly Interpretation

The final system should not only say:

``` text
ANOMALY
```

It should attempt to explain **why** the window was considered unusual.

For an anomalous window, the system can inspect the feature
contributions or compare the window against normal behavior.

For example:

``` text
Anomalous Window
        |
        +-- GPS speed variability ↑
        +-- GPS EPH ↑
        +-- EKF innovation ↑
        +-- roll-rate variability ↑
        +-- actuator output variation ↑
```

This creates a much more useful system for UAV telemetry analysis.

------------------------------------------------------------------------

# 34. Visualization

Future visualizations should include:

### Flight timeline

``` text
time
  ↓
telemetry signal
  ↓
anomaly score
```

### GPS anomaly visualization

Plot:

``` text
GPS speed
GPS uncertainty
satellite count
anomaly score
```

over the same flight timeline.

### Attitude anomaly visualization

Plot:

``` text
roll rate
pitch rate
yaw rate
IMU acceleration
anomaly score
```

### Estimator anomaly visualization

Plot:

``` text
EKF innovations
estimator test ratios
anomaly score
```

This will allow detected anomalies to be inspected against the
underlying flight behavior.

------------------------------------------------------------------------

# 35. Model Comparison

Isolation Forest should be treated as the first baseline rather than the
final model.

Later models can be investigated, depending on dataset size and results:

``` text
Isolation Forest
        ↓
One-Class SVM
        ↓
Local Outlier Factor
        ↓
Autoencoder
        ↓
LSTM / Temporal Autoencoder
```

The purpose of this comparison would be to determine whether a more
complex model provides meaningful improvement over the baseline.

A complex deep-learning model should not be adopted automatically if a
simpler model performs adequately.

------------------------------------------------------------------------

# 36. Evaluation Without Ground-Truth Labels

Because the dataset currently has no anomaly labels, traditional
supervised metrics such as:

``` text
Accuracy
Precision
Recall
F1-score
```

cannot be treated as primary evaluation metrics unless reliable
ground-truth anomaly labels are later obtained.

Instead, evaluation can initially focus on:

-   Stability of anomaly scores.
-   Percentage of windows flagged.
-   Flight-wise consistency.
-   Temporal persistence.
-   Visualization of detected events.
-   Agreement with known telemetry failure indicators.
-   Expert/manual inspection.
-   Synthetic anomaly injection for controlled testing.

If reliable labelled anomaly flights become available later, supervised
evaluation metrics can then be introduced.

------------------------------------------------------------------------

# 37. Synthetic Anomaly Testing

One useful future experiment is to create controlled abnormal signals in
copies of the dataset.

Examples could include:

``` text
GPS speed spike
GPS uncertainty spike
satellite-count drop
battery voltage drop
IMU spike
EKF innovation spike
actuator-output spike
```

The modified data would remain separate from the original dataset.

The objective would be to test whether the anomaly detector can identify
known injected disturbances.

This does **not** replace real anomaly validation, but it can provide a
controlled benchmark when naturally labelled anomalies are limited.

------------------------------------------------------------------------

# 38. Final Planned Architecture

The complete project pipeline is expected to become:

``` text
PX4 ULog Files
       |
       v
Topic / Column Selection
       |
       v
Required-Flight Filtering
       |
       v
Raw Telemetry Extraction
       |
       v
Common 10 Hz Timeline
       |
       v
10-Second Window Assignment
       |
       v
Merged Raw Dataset
       |
       v
Missing-Value Handling
       |
       v
Constant / Uninformative Feature Analysis
       |
       v
EDA
       |
       v
Window-Level Feature Engineering
       |
       v
Feature Validation
       |
       v
Scaling
       |
       v
Isolation Forest Baseline
       |
       v
Anomaly Scores
       |
       v
Temporal Anomaly Events
       |
       v
Visualization + Interpretation
       |
       v
Model Comparison
       |
       v
Evaluation
       |
       v
Final UAV Anomaly Detection System
```

------------------------------------------------------------------------

# 39. Final Objective

The ultimate goal of this dataset is not simply to train a model that
outputs:

``` text
0 = Normal
1 = Anomaly
```

The intended system is a telemetry-analysis pipeline capable of:

1.  Receiving a PX4 ULog flight.
2.  Extracting relevant telemetry.
3.  Synchronizing multi-rate signals.
4.  Representing the flight as temporal windows.
5.  Learning normal flight behavior without requiring predefined anomaly
    labels.
6.  Detecting statistically unusual flight windows.
7.  Grouping anomalous windows into potential anomaly events.
8.  Identifying the telemetry signals associated with the event.
9.  Visualizing the event on the flight timeline.
10. Providing an interpretable explanation of the suspected abnormal
    behavior.

The current `flight_merged_final` dataset is therefore the **foundation
for the machine-learning stage**, not the final output of the project.

------------------------------------------------------------------------

# 40. Current Status

### Completed

-   [x] PX4 ULog preprocessing
-   [x] Topic selection
-   [x] Raw-column selection
-   [x] Required-flight filtering
-   [x] Multi-rate synchronization
-   [x] Common 10 Hz timeline
-   [x] 10-second window architecture
-   [x] Unified raw dataset
-   [x] Missing-value analysis and filling
-   [x] Constant-feature analysis
-   [x] EDA pipeline
-   [x] Final 54-column synchronized dataset

### Next

-   [ ] Validate final CSV programmatically
-   [ ] Build window-level feature engineering
-   [ ] Generate ML feature matrix
-   [ ] Analyze feature distributions
-   [ ] Scale features where appropriate
-   [ ] Train Isolation Forest
-   [ ] Generate anomaly scores
-   [ ] Determine anomaly threshold
-   [ ] Detect persistent anomaly events
-   [ ] Visualize detected anomalies
-   [ ] Interpret anomalous telemetry
-   [ ] Perform flight-wise evaluation
-   [ ] Compare alternative anomaly detection models
-   [ ] Document results and limitations

------------------------------------------------------------------------

## Dataset Principle

> **Keep the raw synchronized telemetry intact first. Perform feature
> engineering and anomaly detection as separate downstream stages.**

This separation is important because it allows the original telemetry
dataset to remain reproducible and auditable while different
feature-engineering and machine-learning approaches can be tested
without rebuilding the entire preprocessing pipeline.
