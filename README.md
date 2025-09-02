# Airline Route Investment Analysis (Q1 2019)

**Goal:** Identify the five U.S. domestic **round‑trip** routes that best balance **on‑time performance** and **rapid recovery of a \$90 M aircraft investment**.

> Motto: **“On time, for you.”** Punctuality is a primary brand promise; ROI discipline is a hard requirement.

---

## ⭐ Executive Summary

* **Method:** Two normalized 0–1 scores per route — **Punctuality** (delay‑based) and **Payback** (time to recoup investment) — combined with equal weights.
* **Top‑5 recommended routes:** `JFK–LAX`, `CLT–FLO`, `JFK–SFO`, `CLT–GSP`, `EWR–SFO`.
* **Why these?** They jointly optimize **reliability** and **payback speed**. JFK trunk routes deliver standout **absolute profit**; Charlotte spokes deliver **high reliability** and **fast break‑even**.

---

## 📚 Business Context & Objectives

* New route launches must justify a **\$90 M per‑aircraft** investment.
* Primary question: **Which 5 routes deliver the strongest combination of reliability and ROI?**
* Key stakeholder priorities:

  1. **On‑time performance** within a 15‑minute free window.
  2. **Fast payback** of capital outlay.

---

## 🧰 Data & Reproducibility

This repo ships a fully reproducible pipeline using **pure, parameterized Python functions**.

### Data Inputs (Q1 2019)

* **Flights** (per‑leg operations & delays)
* **Ticket prices** (itinerary‑level fares)
* **Airports** (IATA metadata)

> **Assumptions** (validated/used in logic):
>
> * Domestic **distance** valid range: `24–5 095 mi`
> * Domestic **airtime** valid range: `8–660 min`
> * **Free delay allowance:** first `15 min` of DEP/ARR delays cost \$0
> * **Aircraft capacity:** `200` seats per leg for passenger‑revenue estimates

### Reproducibility Guarantees

* **Stateless transforms** (no global state; functions work on copies)
* **Deterministic enrichment** via `enrich_and_join_flights()`
* **Pure scoring**: same inputs ⇒ same outputs

---

## 🧪 Data Quality, Cleaning & Munging

#### Ticket Data

* Removed duplicate `ITIN_ID` entries
* Dropped `Passenger` (derived from occupancy rate)
* Filtered `ITIN_FARE` to **\$39–\$10 000** to remove improbable fares

#### Flight Data

* Typed `DISTANCE`, `OCCUPANCY_RATE`, `FL_DATE`
* Excluded cancelled flights `CANCELLED=1`
* Dropped `AIR_TIME < 8` or `> 660` minutes; clipped extreme delays for scoring:

  * `DEP_DELAY < 0 → 0`, `> 300` treated as error/mis‑recorded cancellations
  * `ARR_DELAY < 0 → 0`, `> 300` dropped

#### Airport Data

* Kept **medium/large** airports; dropped invalid `IATA_CODE`

#### Enrichment Pipeline: `enrich_and_join_flights()`

* Strips/uppercases `ORIGIN`, `DESTINATION`
* Normalizes ticket `TRIP_NAME` → **sorted uppercase pair** (e.g., `CLT–FLO`)
* Extracts `IATA_CODE → AIRPORT_TYPE`
* Joins `ORIGIN_AIRPORT_TYPE`, `DESTINATION_AIRPORT_TYPE`
* Builds leg‑level `TRIP_NAME`; merges **avg ticket price** per route

> The function always produces identical enriched outputs on identical inputs and can be re‑run seamlessly on updated data.

---

## 📏 Scoring Methodology

All scoring is implemented as composable, side‑effect‑free functions.

```python
compute_punctuality_score(df, delay_threshold=15.0)
```

* Clips early departures/arrivals to 0 late minutes; computes a penalty vs. free window
* `punctuality_score = max(0, 1 - penalty)` in **\[0, 1]**

```python
compute_payback_period(df, investment_per_route=90_000_000, profit_period_months=3.0)
```

* `monthly_profit = TOTAL_PROFIT / profit_period_months`
* `payback_months = investment_per_route / monthly_profit`

```python
score_payback(df)
```

* Normalizes by the slowest route:
  `payback_score = max(0, 1 - payback_months / max_payback)` in **\[0, 1]**

```python
recommend_routes(df, n=5, invest_per_route=90_000_000,
                 profit_period_months=3.0,
                 weight_punctuality=0.5, weight_payback=0.5)
```

* Applies the three steps above
* `combined_score = 0.5 * punctuality + 0.5 * payback`
* Returns **top‑n** routes by `combined_score`

> All thresholds, weights, and investment figures are **parameters**, making the analysis easy to tune.

---

## 🧮 Break‑Even & Financial Logic

* **Profit per round‑trip**: `Q1_TOTAL_PROFIT ÷ (leg_count / 2)`
* **Break‑even round‑trips**: `90 M ÷ profit_per_round_trip`

**Highlights**

* `CLT–FLO` needs **\~391** trips to break even — exceptional per‑trip yield.
* `JFK–LAX` needs **\~1 977** trips — huge absolute profit but capital‑heavy.
* `ATL–CLT` needs **\~3 595** trips — smaller per‑trip profit extends payback.

---

### 1) Environment

```bash
# Using conda (recommended)
conda create -n route-invest python=3.11 -y
conda activate route-invest

# Or plain pip\python -m venv .venv
source .venv/bin/activate  # Windows: .venv\\Scripts\\activate

pip install -r requirements.txt
```

**`requirements.txt` (example)**

```
pandas
numpy
matplotlib
seaborn
geopandas
shapely
pyproj
jupyter
```

### 2) Run Analysis (script)

```
python file
  --flights data/flights_q1_2019.csv \
  --tickets data/ticket_prices_q1_2019.csv \
  --airports data/airports.csv \

```

This will:

1. Enrich & join inputs (`enrich_and_join_flights()`)
2. Score punctuality & payback
3. Export **top‑N recommendations** and **figures** to `outputs/` & `figures/`

### 3) Run Notebook

Open `notebooks/01_q1_2019_route_investment.ipynb` and run all cells.


---

## 🧾 Data Dictionary (selected derived fields)

| Field               | Type  | Description                                            |
| ------------------- | ----- | ------------------------------------------------------ |
| `TRIP_NAME`         | str   | Alphabetical route key, e.g., `CLT–FLO`                |
| `punctuality_score` | float | On‑time metric in `[0, 1]` vs. 15‑min free window      |
| `payback_months`    | float | Months to recoup \$90 M given observed profit run‑rate |
| `payback_score`     | float | Normalized speed in `[0, 1]` (1 = fastest)             |
| `combined_score`    | float | Weighted mean of punctuality & payback                 |

---

## 📈 KPIs & Success Metrics (for production dashboards)

1. **Operations**: On‑time rate, avg delay minutes, cancellation/diversion rates
2. **Capacity & Load**: Load factor, break‑even trips remaining, ancillary per pax
3. **Financial**: RASM, CASM, maintenance cost/flight
4. **Customer**: NPS, on‑time rate, ancillary per pax
5. **Maintenance & Reliability**: maintenance \$/flight, cancels/diversions, avg delay minutes

---

## 🛣️ Roadmap

* Add **frequency**, **fuel‑price variance**, **passenger mix** to the model
* Build an **interactive dashboard** for scenario analysis
* Sensitivity analysis on **delay threshold** & **weighting**
* Auto‑generation of **geo maps** for top‑N routes




