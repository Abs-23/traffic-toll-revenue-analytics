# Traffic and Toll Revenue Forecasting – Databricks and Power BI

<img width="1279" height="724" alt="Traffic   Toll Revenue Scenario" src="https://github.com/user-attachments/assets/529ee80c-b010-4564-825c-4afc5a17f77f" />



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

├─ data/
│ ├─ Metro_Interstate_Traffic_Volume.csv # Original hourly traffic and weather data
│ └─ traffic_daily_export.csv # Daily model-ready export used by Power BI
├─ notebooks/
│ └─ traffic_toll_revenue_forecasting.py # Databricks notebook with medallion, modeling, and scenarios
├─ reports/
│ ├─ Traffic_Toll_Revenue_Scenarios.pbix # Power BI report
│ └─ traffic_toll_revenue_dashboard.png # Screenshot used in this README
└─ README.md # Project documentation

```

---

## 7. How to Run

### 7.1 Databricks

1. Create a catalog and the `raw`, `silver`, and `gold` schemas in Unity Catalog.  
2. Create a volume and upload `Metro_Interstate_Traffic_Volume.csv` from the `data` folder.  
3. Import `notebooks/traffic_toll_revenue_forecasting.py` into Databricks.  
4. Attach a cluster and run the notebook to:
   - Ingest raw data into the bronze table.  
   - Build the silver hourly and gold daily tables.  
   - Train and evaluate the forecasting model.  
   - Generate the daily export used by Power BI, if needed.

### 7.2 Power BI

1. Open `reports/Traffic_Toll_Revenue_Scenarios.pbix` in Power BI Desktop.  
2. Update the data source to point to your local `traffic_daily_export.csv` if the path has changed.  
3. Interact with the slicers and visuals to explore:
   - Daily actual vs forecast traffic.  
   - Annual base vs scenario revenue.  
   - Weekday vs weekend differences.

---

## 8. Intended Audience

This project is aimed at:

- **Traffic and revenue analysts** who want a concrete example of how to connect traffic forecasting with toll pricing scenarios.  
- **Data scientists and data engineers** who want to see a concise medallion architecture on Databricks feeding a BI tool.  
- **Hiring managers** who want evidence of end‑to‑end ownership, from raw data and modeling decisions to revenue‑focused scenario analysis and executive‑style reporting.

It is not a production‑grade Traffic and Revenue model. It is a transparent, reproducible case study that demonstrates the full workflow and highlights the trade‑offs involved in toll pricing decisions.
