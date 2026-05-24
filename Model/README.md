# New-York-City-Ride-Hailing-Market-Share-Prediction-Engine
# NYC Rideshare Market Share Prediction Model

This model integrates temporal, geographic, business, and meteorological dimensions to capture the dynamic competition between High-Volume For-Hire Vehicles (HVFHV, e.g., Uber/Lyft) and traditional NYC Taxis (Yellow/Green). 

The following documentation details the feature engineering process, variable definitions, and the underlying business logic used to simulate 2026 market conditions.

---

## 1. Core Target Variable
* **Variable Name**: `market_share`
* **Generation**: 
    * Aggregated from raw trip data. HVFHV counts are adjusted by a restoration multiplier (e.g., 12.5x) based on the sampling rate.
    * **Formula**: $\text{HVFHV Volume} / (\text{HVFHV Volume} + \text{Taxi Volume})$.
* **Application**: The model applies a **Logit Transformation** to map the share from $[0, 1]$ to a real number space during training, and uses a **Sigmoid Function** to revert predictions back to probabilities. This ensures stable boundary performance.
* **Real-world Significance**: Serves as the primary KPI for competitive analysis and resource allocation in the urban mobility sector.

---

## 2. Temporal & Geographic Features
* **Zone (Pickup Location)**:
    * **Generation**: Mapped from official NYC `PULocationID` to specific neighborhood names. Specially, I treated the Zone as a categorical feature in CatBoost.
    * **Significance**: Represents the socio-economic "DNA" of a neighborhood (e.g., Commercial, Residential, Industrial), which dictates baseline travel patterns.
* **Hour / DayOfWeek / IsWeekend**:
    * **Generation**: Extracted from pickup timestamps.
    * **Significance**: Captures the rhythm of city life, distinguishing between rush-hour commutes, late-night social activities, and weekend tourism.
* **Month**:
    * **Generation**: Extracted from the calendar month.
    * **Significance**: Reflects seasonality, holiday effects, and long-term climate-driven shifts in travel behavior.

---

## 3. Business & Trip Features
* **trip_miles (Average Distance)**:
    * **Generation**: The mean trip distance for HVFHVs within a specific spatio-temporal window.
    * **Significance**: Distinguishes between long-haul airport/inter-borough trips and short-range community errands.
* **speed_mph (Average Speed)**:
    * **Generation**: Calculated as `trip_miles / (trip_time / 3600)`.
    * **Significance**: A direct proxy for urban congestion. Market share dynamics shift significantly during gridlock due to different pricing structures (dynamic vs. metered).
* **cost_per_mile (Unit Cost)**:
    * **Generation**: `base_passenger_fare / trip_miles`.
    * **Significance**: Measures price competitiveness, a primary driver for user mode choice.

---

## 4. Meteorological Features
* **temp_c (Temperature)**:
    * **Generation**: Historical weather data retrieved via the Open-Meteo API.
    * **Significance**: Extreme temperatures (heatwaves or deep freezes) typically increase user preference for "door-to-door" rideshare services over street-hailing.
* **is_extreme_weather (Extreme Weather Flag)**:
    * **Generation**: A binary feature triggered by temperature thresholds ($\le 0^\circ\text{C}$ or $\ge 32^\circ\text{C}$) or precipitation ($> 2.5\text{mm}$).
    * **Significance**: During heavy rain or snow, street-hail availability plummets, driving users toward the high-certainty booking environment of rideshare apps.

---

## 5. Autoregressive Time-Series Features
These features provide the model with "short-term memory," often yielding the highest predictive power.
* **lag_1h / 2h / 24h_share**:
    * **Generation**: The actual market share from the same zone 1 hour, 2 hours, or 24 hours prior.
    * **Significance**: Captures temporal inertia—if rideshare is dominating an area an hour ago, it is likely to remain dominant now.
* **rolling_3h_mean_share**:
    * **Generation**: The 3-hour moving average of market share.
    * **Significance**: Smooths out transient noise to identify emerging trends in the market.
* **rolling_6h_max_share**:
    * **Generation**: The maximum market share observed within the last 6 hours.
    * **Significance**: Identifies the "aftershocks" of high-demand pulses and sets the upper-bound expectations for a specific time window.

---

## Technical Summary
The model utilizes a dual-tower ensemble of **LightGBM** and **CatBoost**. To simulate future performance in 2026, the model is trained exclusively on 2025 data, using an **Out-of-Time (OOT) validation** strategy where the final months of 2025 are held back for evaluation. This ensures the model adapts to modern market structures rather than outdated historical patterns from 2024.
