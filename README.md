# System Model Specification: Indoor Air Quality (PM2.5) Digital Twin

- **Project Title:** Digital Twins – Indoor Air Quality ($\text{PM}_{2.5}$) Simulation & Synchronization
- **Course:** Simulations – Digital Twin
- **Team:** Team A – Frendl Krisztián & Kozicz Gergő
- **Prototype Category:** Monitoring, Reporting, and Verification (MRV) Digital Twin (Historical Replay, Parameter Calibration & Live Context Transfer)

---

## 1. Modelling Purpose & Scope

The primary objective of this Digital Twin prototype is to monitor, simulate, synchronize, calibrate, and predict indoor fine particulate matter ($\text{PM}_{2.5}$) concentrations within a residential room under dynamic environmental and operational conditions.

- **Core Modelling Question:** How does indoor $\text{PM}_{2.5}$ concentration ($PM_{\text{in}}$) evolve over time under varying outdoor air pollution ($PM_{\text{out}}$), wind speed ($v$), and relative humidity ($h$)?
- **Closed-Loop State Synchronization (Week 4):** How effectively does chronological replay-mode proportional state updating ($K = 0.5$) eliminate model drift and respond to unmodeled indoor emission spikes (such as cooking) compared to open-loop simulation?
- **Empirical Parameter Calibration (Week 5):** How do 1-parameter (`k_base`) and 2-parameter (`k_base`, `c_h`) grid-search calibrations improve generalization accuracy (RMSE) on an unseen chronological test period?
- **Live Data Integration & Context-Based Scaling (Week 6):** How can the historically calibrated physical model be transferred via rule-based, non-AI scaling to estimate current indoor $\text{PM}_{2.5}$ using live outdoor APIs (OpenAQ v3 and Open-Meteo) near Weslaco, Texas?
- **What-If Scenario Analysis & Decision Support:** How do changes in building airtightness (Scenario A), artificial humidification (Scenario B), internal cooking emissions (Scenario C), active air purification (Scenario D), and stormy drafts with filtration (Scenario E) impact indoor exposure?

---

## 2. System Boundary & Physical Decomposition

The system model separates the indoor micro-environment from external meteorological, environmental, and behavioral drivers:

| Boundary Domain | Included Elements & Processes |
| :--- | :--- |
| **Inside the System (Indoor Room Volume)** | • **Indoor Air State:** Current indoor $\text{PM}_{2.5}$ concentration $PM_{\text{in}}(t)$<br>• **Dynamic Infiltration:** Wind-driven outdoor-to-indoor particle penetration $k(v) \cdot (PM_{\text{out}} - PM_{\text{in}})$<br>• **Gravitational Deposition:** Humidity-dependent particle growth and settling $\gamma(h) \cdot PM_{\text{in}}$<br>• **Internal Emissions:** Occupant-generated pollution $\alpha_{\text{source}}$<br>• **Active Filtration:** Clean Air Delivery Rate (CADR) removal via air purifier $\beta_{\text{purifier}}$ |
| **Outside the System (External Drivers)** | • **Meteorological & Pollution Disturbances:** Outdoor $\text{PM}_{2.5}$ ($PM_{\text{out}}$), wind speed ($v$), and ambient relative humidity ($h$) from historical logs or live REST APIs<br>• **Occupant Activity Schedule:** Unmeasured or scheduled cooking, cleaning, and occupancy events<br>• **Control Commands:** Air purifier CADR ($\beta_{\text{purifier}}$) and humidifier settings |
| **System Outputs** | • Hourly open-loop simulated, synchronized ($C_{\text{updated}}$), and one-step-ahead predicted indoor $\text{PM}_{2.5}$ trajectories in $\mu\text{g/m}^3$<br>• Quantitative validation metrics (MAE, RMSE) and historical-to-current indoor $\text{PM}_{2.5}$ estimates<br>• Multi-scenario comparative exposure trajectories |

---

## 3. System Variables & Parameters

All system quantities follow the Digital Twin state-space framework, incorporating both the original baseline parameters and the empirically calibrated Week 5 values:

| Category | Symbol | Code Variable | Unit | Initial Baseline | Calibrated (Week 5) | Description |
| :--- | :---: | :--- | :---: | :---: | :---: | :--- |
| **State ($x_t$)** | $PM_{\text{in}}(t)$ / $C_t$ | `PM_in` / `C_sim` | $\mu\text{g/m}^3$ | $C_{\text{obs}}(0)$ | $C_0$ (e.g., $3.96$) | Indoor $\text{PM}_{2.5}$ concentration carrying system memory across time steps. |
| **Control ($u_t$)** | $\beta_{\text{purifier}}$ | `beta_purifier` | $\mu\text{g}/(\text{m}^3\cdot\text{h})$ | `0.0` | `0.0` | Active air purifier filtration rate (CADR removal). |
| **Control ($u_t$)** | $\alpha_{\text{source}}$ | `alpha_source` | $\mu\text{g}/(\text{m}^3\cdot\text{h})$ | `0.0` | `0.0` | Controllable/schedulable indoor pollution generation (e.g., cooking). |
| **Disturbance ($d_t$)** | $PM_{\text{out}}(t)$ | `pm25` | $\mu\text{g/m}^3$ | Time-series | Live API | Uncontrolled outdoor $\text{PM}_{2.5}$ concentration entering via infiltration. |
| **Disturbance ($d_t$)** | $v(t)$ | `wind` | $\text{m/s}$ | Time-series | Live API | Outdoor wind speed increasing pressure across the building envelope. |
| **Disturbance ($d_t$)** | $h(t)$ | `rh` | $\%$ | Time-series | Live API | Relative humidity influencing hygroscopic particle growth and deposition. |
| **Parameter ($\theta$)** | $k_{\text{base}}$ ($a$) | `k_base` | $\text{h}^{-1}$ | `0.1000` | **`0.0010`** | Base building envelope air exchange / infiltration rate in calm conditions ($v = 0$). |
| **Parameter ($\theta$)** | $c_w$ | `c_w` | $(\text{m/s})^{-1}\cdot\text{h}^{-1}$ | `0.05` | `0.05` (fixed) | Wind sensitivity coefficient for dynamic air infiltration. |
| **Parameter ($\theta$)** | $c_h$ ($k_{\text{dep}}$) | `c_h` | $\text{h}^{-1}\cdot\%^{-1}$ | `0.00100` | **`0.00125`** | Humidity-dependent deposition coefficient characterizing particle settling. |
| **Parameter ($\theta$)** | $K$ | `K` | — | `0.5` | `0.5` | Proportional synchronization gain blending simulated state with indoor sensor telemetry. |
| **Parameter ($\theta$)** | $\Delta t$ | `delta_t` | $\text{h}$ | `1.0` | `1.0` | Discrete hourly simulation and synchronization time step. |
| **Observation ($y_t$)** | $C_{\text{obs}}(t)$ | `pm25_in` | $\mu\text{g/m}^3$ | Measured | N/A (live transfer) | Ground-truth indoor $\text{PM}_{2.5}$ sensor reading (available in historical replay). |

---

## 4. Mathematical & Behavioral Formulation

The core physical engine implements a discrete-time **Extended Mass Balance Equation** over an hourly time step ($\Delta t = 1\text{ h}$).

### 4.1. Dynamic Physical Coefficients
At each hourly step $t$, the dynamic infiltration rate $k(v_t)$ and humidity-dependent deposition rate $\gamma(h_t)$ are computed from boundary inputs:

$$k(v_t) = k_{\text{base}} + c_w \cdot v_t$$

$$\gamma(h_t) = c_h \cdot h_t$$

### 4.2. Open-Loop State-Space Transition (`simulate_advanced_indoor_aq`)
The hourly concentration update is governed by infiltration, deposition, internal generation, and active filtration, with a non-negativity constraint:

$$\Delta PM_{\text{in}}(t) = \Delta t \cdot \left[ k(v_t) \cdot \left(PM_{\text{out}}(t) - PM_{\text{in}}(t-1)\right) - \gamma(h_t) \cdot PM_{\text{in}}(t-1) + \alpha_{\text{source}} - \beta_{\text{purifier}} \right]$$

$$PM_{\text{in}}(t) = \max\left(0,\; PM_{\text{in}}(t-1) + \Delta PM_{\text{in}}(t)\right)$$

### 4.3. Chronological Replay Synchronization (`simulate_synchronized_aq`)
To prevent cumulative open-loop drift, the Digital Twin executes a closed-loop **Observe $\rightarrow$ Synchronize $\rightarrow$ Predict $\rightarrow$ Validate** cycle at each hourly step using proportional state updating ($K = 0.5$):

1. **Observe:** Read the current indoor sensor measurement $C_{\text{obs}}(t)$ (`pm25_in`) and outdoor boundary inputs (`pm25`, `wind`, `rh`).
2. **Synchronize:** Blend the prior simulated state $C_{\text{sim}}(t)$ with the real observation:
   $$C_{\text{updated}}(t) = C_{\text{sim}}(t) + K \cdot \left(C_{\text{obs}}(t) - C_{\text{sim}}(t)\right)$$
3. **Predict Next Step:** Advance $C_{\text{updated}}(t)$ through the unchanged mass-balance transition equation to compute and store the one-step-ahead prediction before reading the next observation.
4. **Validate & Repeat:** Record the step-by-step error against the observation and advance to $t+1$.

---

## 5. Data Catalogue & Integration Pipeline

### 5.1. Data Sources

| Source Name | Provider | Access Method | Variables | Units | Temporal Coverage & Resolution | Spatial Coverage | Limitations / Missing Data |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **Historical Indoor & Outdoor Air Quality** | Mendeley Data (`doi: 10.17632/yk2pz4hcb5.2`) | Local CSV (`Home1_Weslaco_Indoor_Outdoor_updated.csv`) | Indoor $\text{PM}_{2.5}$, Outdoor $\text{PM}_{2.5}$, Indoor/Outdoor $\text{RH}$ | $\mu\text{g/m}^3$, $\%$ | June–July 2024 (1-hour average) | Home-1, Weslaco, TX | Static historical dataset; no real-time updates; indoor source $S(t)$ and occupancy unmeasured. |
| **Historical Wind Data** | TCEQ CAMS C1023 | Local CSV (`TCEQ_CAMSdata_WindRose.csv`) | Resultant Wind Speed ($\text{RWS}$) | $\text{m/s}$ | June–July 2024 (1-hour average) | Harlingen, TX (~20 km from Weslaco) | Regional monitoring station; slight spatial mismatch vs. exact residence. |
| **Current Outdoor $\text{PM}_{2.5}$** | OpenAQ API v3 (`api.openaq.org`) | REST API (`OPENAQ_API_KEY`) | Outdoor $\text{PM}_{2.5}$ | $\mu\text{g/m}^3$ | Recent 72-hour rolling window (1-hour average) | Nearest station within 25 km of Weslaco, TX (e.g., *Edinburg C1046*, Sensor ID `2071343`) | Occasional missing hourly intervals or sensor dropouts; requires API key. |
| **Current Weather Context** | Open-Meteo (`api.open-meteo.com`) | REST API (No key required) | `relative_humidity_2m`, `wind_speed_10m` | $\%$, $\text{m/s}$ | Recent 72-hour rolling window (1-hour average, UTC) | Weslaco, TX ($26.1595^\circ\text{N},\; -97.9908^\circ\text{W}$) | Gridded meteorological model estimate rather than on-site physical weather sensor. |

### 5.2. Automated Preprocessing & Harmonization
- **Historical Pipeline:** Parses and aligns timestamps across indoor, outdoor, and wind CSV tables; merges on `datetime` (732 initial merged records); drops missing values (`NaN`, 6 records in `pm25`) and duplicate timestamps (`keep='first'`); removes negative/invalid physical readings; sorts and verifies strict monotonic chronological order (**726 cleaned hourly records × 6 columns**).
- **Live API Pipeline:** Retains raw copies (`raw_openaq_df`, `raw_weather_df`) separate from processed data; floors timestamps to hourly UTC (`dt.floor("h")`); performs an inner merge on `datetime`; removes missing/duplicate/negative values to build a model-ready dataset (e.g., **48 harmonized hourly records × 4 columns**).

---

## 6. Synchronization, Calibration & Context-Transfer Results

### 6.1. Week 4: Closed-Loop Synchronization Evaluation
Evaluated over a 72-hour chronological replay window starting `2024-06-16 00:00:00` (initial $PM_{\text{in}} = 4.38\ \mu\text{g/m}^3$) using baseline parameters (`k_base = 0.1`, `c_w = 0.05`, `c_h = 0.001`) and $K = 0.5$:

| Evaluation Comparison | Mean Absolute Error (MAE) |
| :--- | :---: |
| **Observed vs. Simulated (Open-Loop)** | `1.4662 µg/m³` |
| **Observed vs. Synchronized State ($K = 0.5$)** | `0.4937 µg/m³` |
| **Observed vs. One-Step-Ahead Prediction ($K = 0.5$)** | `0.8095 µg/m³` |

- **Key Finding:** Open-loop execution drifts over time and cannot detect unmodeled internal events (such as kitchen cooking spikes up to $13.94\ \mu\text{g/m}^3$ when outdoor $\text{PM}_{2.5}$ is $6.70\text{–}7.88\ \mu\text{g/m}^3$). Closed-loop synchronization ($K = 0.5$) substantially lowers prediction error and reacts immediately to indoor anomalies.

### 6.2. Week 5: Model Parameter Calibration
Using a strict chronological split—**Training Set:** first 336 hours (2 weeks, `df.iloc[:336]`) and **Unseen Test Set:** next 96 hours (4 days, `df.iloc[336:432]`)—parameters were calibrated via grid search (`k_base` $\in [0.001, 2.0]$ across 30 points; `c_h` $\in [0.0, 0.03]$ across 25 points) with $K = 0.5$ and `c_w = 0.05` fixed:

| Parameter / Metric | Initial (Original) | Calibrated (1-Param) | Calibrated (2-Param) |
| :--- | :---: | :---: | :---: |
| **Air exchange: `k_base` ($a$)** | `0.1000` | `0.0010` | `0.0010` |
| **Deposition: `c_h` ($k_{\text{dep}}$)** | `0.00100` | `0.00100` (fixed) | `0.00125` |
| **Training RMSE ($\mu\text{g/m}^3$)** | `1.8023` | `1.4414` | **`1.4314`** |
| **Unseen Test RMSE ($\mu\text{g/m}^3$)** | `1.5630` | `1.3866` | **`1.3838`** |

- **Key Finding:** The uncalibrated model overestimates indoor $\text{PM}_{2.5}$ because baseline infiltration is too high relative to indoor deposition. Joint 2-parameter calibration balances air exchange (`k_base = 0.0010`) and humidity-driven particle settling (`c_h = 0.00125`), yielding the lowest RMSE on both training and unseen test sets.

### 6.3. Week 6: Historical-to-Current Context Transfer
- **Rule-Based Scaling (Non-AI):** The calibrated Week 5 parameters (`k_base = 0.0010 h⁻¹`, `c_h = 0.00125 h⁻¹·%⁻¹`, `c_w = 0.05`) are directly reused on live harmonized OpenAQ + Open-Meteo data near Weslaco, TX.
- **Initialization & Operational Context:** Without live indoor telemetry, the simulation is initialized with an explicit estimated starting concentration $C_0$ at $t = 0$ (e.g., $C_0 = 3.96\ \mu\text{g/m}^3$ in the sample run) and assumes unchanged residential geometry under baseline unoccupied/no-cooking conditions ($\alpha_{\text{source}} = 0.0$, $\beta_{\text{purifier}} = 0.0$).

---

## 7. What-If Scenarios & Assumptions

### 7.1. Configured What-If Simulation Regimes

| Scenario ID | Scenario Name | `k_base` | `c_w` | `c_h` | `alpha_source` | `beta_purifier` | Physical Interpretation |
| :--- | :--- | :---: | :---: | :---: | :---: | :---: | :--- |
| **Baseline** | Normal Conditions | `0.1` | `0.05` | `0.001` | `0.0` | `0.0` | Standard insulated room with no internal sources or active filtration. |
| **Scenario A** | Drafty House | `0.3` | `0.10` | `0.001` | `0.0` | `0.0` | Poor insulation and high wind sensitivity driving rapid outdoor ingress. |
| **Scenario B** | Humidifier ON | `0.1` | `0.05` | `0.005` | `0.0` | `0.0` | Elevated humidity deposition accelerating gravitational particle settling. |
| **Scenario C** | Cooking Event | `0.1` | `0.05` | `0.001` | `12.0` | `0.0` | Strong continuous internal pollution source without air purification. |
| **Scenario D** | Cooking + Purifier | `0.1` | `0.05` | `0.001` | `12.0` | `15.0` | Active air purifier ($\beta > \alpha$) mitigating cooking emissions. |
| **Scenario E** | Stormy Draft | `0.5` | `0.20` | `0.001` | `12.0` | `15.0` | Stress test: extreme wind infiltration and cooking against active filtration. |

### 7.2. Assumptions & Limitations
- **Well-Mixed Single-Zone Approximation:** Assumes uniform indoor particulate concentration across the room volume.
- **Spatial & Sensor Proxies:** Historical wind data (Harlingen CAMS C1023) and live outdoor $\text{PM}_{2.5}$ (Edinburg C1046) come from regional stations rather than on-site sensors; Open-Meteo provides gridded weather estimates.
- **Unobserved Live Indoor Context:** Exact room volume ($V$), real-time occupancy, window-opening events, and active cooking emissions ($S(t)$) are unavailable during current-period context transfer, limiting live estimates to baseline infiltration-deposition dynamics unless paired with a physical indoor sensor.

---

## 8. Environment & Usage

- **Dependencies:** `pandas`, `numpy`, `matplotlib`, `scikit-learn`, `requests`
- **Historical Data Paths:**
  - `../data/Home1_Weslaco_Indoor_Outdoor_updated.csv`
  - `../data/TCEQ_CAMSdata_WindRose.csv`
- **Environment Variable (for Live OpenAQ v3 Integration):**
  ```bash
  export OPENAQ_API_KEY="your_openaq_v3_api_key"
