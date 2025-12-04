# Traffic and Toll Revenue Forecasting – Databricks and Power BI

<img width="1279" height="729" alt="Traffic   Toll Revenue Scenario" src="https://github.com/user-attachments/assets/e80be25b-04ec-4f14-8d08-6e8f442cd8e4" />


This project uses a Databricks medallion architecture, Python, and Power BI to forecast daily highway traffic and evaluate toll pricing strategies, including a simple demand‑response (elasticity) assumption.

The aim is to move from raw traffic and weather data to governed, model‑ready tables and then to revenue‑focused decision support that a business stakeholder can actually use.

---

## 1. Power BI Dashboard Overview

The Power BI report (`reports/Traffic_Toll_Revenue_Scenarios.pbix`) is designed as an executive‑style summary of the analysis.

The main page includes:

- **KPI cards** that show base annual toll revenue, weekend‑surcharge scenario revenue, revenue uplift percentage, and average daily traffic, all controlled by a year slicer.  
- A **daily traffic line chart** comparing actual volume and model forecast, which highlights seasonality and gives an intuitive sense of model fit.  
- An **annual revenue column chart** that compares base vs weekend pricing scenario by year, making revenue impacts easy to interpret.  
- **Weekday vs weekend charts** that compare average daily traffic and revenue, helping to justify differentiated tolling strategies.

The screenshot above comes directly from this report to give an immediate visual impression of the project when someone opens the repository.

---

## 2. Problem Overview

Highway operators and concessionaires need to understand how traffic volume and toll pricing interact so they can plan revenues and manage congestion.

This project:

- Forecasts **daily traffic volumes** from historical traffic and weather data.  
- Translates those forecasts into **toll revenue** under a base toll policy and a weekend‑surcharge scenario.  
- Introduces a simple **price elasticity of demand** assumption so higher tolls lead to a realistic reduction in traffic.  
- Surfaces the results through a **Power BI dashboard** that supports interactive exploration by year and day type.

The focus is less on building a highly complex model and more on connecting a reasonable forecast to clear revenue impacts.

---

## 3. Data and Architecture

### 3.1 Source Data

The project uses the Metro Interstate Traffic Volume dataset (hourly traffic counts and related weather features).

The raw CSV is stored in `data/Metro_Interstate_Traffic_Volume.csv` and includes fields such as:

- `date_time`  
- `traffic_volume`  
- `holiday`  
- `temp`, `rain_1h`, `snow_1h`, `clouds_all`  
- `weather_main`, `weather_description`  

A daily export used by Power BI is stored as `data/traffic_daily_export.csv`.

### 3.2 Medallion Architecture (Databricks Lakehouse)

The project follows a **bronze / silver / gold** pattern in Databricks using Unity Catalog.

- **Bronze – Raw ingest**  
  - Table: `raw.metro_interstate_raw`  
  - Content: a direct load of the hourly CSV with minimal transformations.  
  - Purpose: preserve the raw traffic and weather data exactly as provided.

- **Silver – Clean and feature‑rich hourly table**  
  - Table: `silver.metro_interstate_hourly`  
  - Key transformations:
    - Add time‑based features: `hour_of_day`, `day_of_week`, `month`, `is_weekend`.  
    - Retain weather and holiday fields: `holiday`, `temp`, `rain_1h`, `snow_1h`, `clouds_all`, `weather_main`.  
  - Purpose: create a clean, feature‑rich hourly dataset that is easy to aggregate and analyze.

- **Gold – Daily model‑ready table**  
  - Table: `gold.metro_interstate_daily`  
  - Key transformations:
    - Aggregate to daily level: `total_traffic_volume`, `avg_temp`, `max_rain_1h`, `max_snow_1h`, `avg_clouds_all`.  
    - Preserve relevant context: `date`, `year`, `month`, `is_weekend`, `any_holiday`.  
  - Purpose: provide a **daily**, model‑ready table for forecasting and revenue scenarios.

All transformation logic is contained in the Databricks notebook under `notebooks/traffic_toll_revenue_forecasting.py`.

---

## 4. Modeling Approach

The modeling is deliberately straightforward and focused on supporting pricing and revenue questions instead of maximizing benchmark scores.

### 4.1 Features and Target

From the gold table:

- **Target**  
  - `total_traffic_volume` (daily total vehicles)

- **Example features**  
  - Time features: `month`, `is_weekend`  
  - Weather features: `avg_temp`, `max_rain_1h`, `max_snow_1h`, `avg_clouds_all`

### 4.2 Model

- Library: `scikit-learn`  
- Algorithm: `RandomForestRegressor`  
- Train / test split: time‑ordered, using the first 80 percent of dates for training and the remaining 20 percent for testing.

Using a time‑based split mirrors real‑world usage where we always forecast **future** from **past** and avoid leaking information through random shuffling.

### 4.3 Performance

- A first pass using only weather features produced a high error and a negative R².  
- Adding basic time features such as `month` and `is_weekend` improved performance to approximately:
  - **MAE:** ~19,000 vehicles per day  
  - **R²:** ~0.08  

The model is not meant as a production‑ready forecasting system. It is sufficient to:

- Demonstrate how engineered features change accuracy.  
- Provide a plausible base for **scenario analysis** of toll policies.

---

## 5. Toll Revenue Scenarios and Elasticity

### 5.1 Base Toll Revenue

Starting from predicted daily volume, the project:

- Assumes a simple vehicle mix, for example 85 percent cars and 15 percent trucks.  
- Assigns baseline tolls per trip, for example 3 units for cars and 7 units for trucks.  
- Computes **base revenue** per day and then aggregates to the annual level.

### 5.2 Weekend Toll Surcharge Scenario

To explore pricing strategy, a weekend‑surcharge scenario is defined:

- Weekend tolls are increased by **20 percent** for both cars and trucks.  
- Weekday tolls remain at the base level.  
- The scenario is first evaluated assuming unchanged volume, and then volume is adjusted using elasticity.

### 5.3 Demand Response (Elasticity)

To avoid unrealistically assuming that volume stays constant when prices change, a simple elasticity is introduced:

- Price elasticity of demand: **−0.2**.  
- Interpretation: a 10 percent toll increase leads to roughly a 2 percent drop in demand.  
- For a 20 percent weekend toll increase, weekend volumes are scaled down accordingly.

The adjusted volumes are used together with the higher weekend tolls to compute **scenario revenue** that reflects both price and demand changes.

### 5.4 Indicative Result

Under this simplified framework, a 20 percent weekend toll increase with an elasticity of −0.2 yields:

- A modest reduction in weekend traffic volume.  
- A **net increase** in projected annual toll revenue of roughly **3–4 percent** in several recent years.

The exact uplift depends on the data and can be recomputed by running the notebook.

---

## 6. Repository Structure

