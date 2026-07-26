#  Football Analytics BI

> End-to-end football analytics project featuring ETL pipelines, PostgreSQL, SQL analytics, Power BI dashboards, and machine learning.

**Python • PostgreSQL • SQL • Power BI • ETL • Machine Learning**

<p align="left">

![Python](https://img.shields.io/badge/Python-3.11-blue?logo=python)
![PostgreSQL](https://img.shields.io/badge/PostgreSQL-Database-blue?logo=postgresql)
![Power BI](https://img.shields.io/badge/Power%20BI-Dashboard-yellow?logo=powerbi)
![SQL](https://img.shields.io/badge/SQL-Analytics-orange)
![MIT License](https://img.shields.io/badge/License-MIT-green)

</p>

---

## Project Overview

This project demonstrates a complete data engineering and analytics workflow applied to professional football data:

**Extract** raw match, team, and standings data from the Football-Data.org API  
**Transform** and clean it using pandas with production-grade error handling  
**Load** it into a normalized PostgreSQL star schema  
**Analyze** it with advanced SQL (CTEs, window functions, aggregations)  
**Visualize** it in a multi-page Power BI dashboard  
**Predict** match outcomes using a Random Forest classifier

---

---

# 🏗️ Architecture

```text
Football-Data.org API
            │
            ▼
      Python ETL Pipeline
            │
            ▼
     Data Transformation
            │
            ▼
   PostgreSQL Data Warehouse
            │
            ▼
      Advanced SQL Queries
            │
            ├──────────────┐
            ▼              ▼
    Power BI Dashboard   Machine Learning
            │              │
            └──────┬───────┘
                   ▼
          Business Insights
```

The project follows a complete end-to-end analytics workflow, starting from data extraction through the Football-Data.org API, transforming and loading data into a PostgreSQL data warehouse, performing SQL-based analytics, visualizing insights in Power BI, and applying machine learning for match outcome prediction.

## Project Structure

```
football_analytics/
│
├── data/
│   ├── raw/           # Raw JSON responses from API (auto-created)
│   └── processed/     # Processed intermediate files
│
├── sql/
│   ├── 01_schema.sql  # Full PostgreSQL schema with indexes + constraints
│   └── 02_analytics_queries.sql   # 10 production SQL analytics queries
│
├── notebooks/         # Jupyter EDA notebooks (add your own)
│
├── src/
│   ├── api/
│   │   └── football_data_client.py    # Rate-limited API client
│   ├── etl/
│   │   ├── transform.py               # Data cleaning + normalization
│   │   ├── load.py                    # PostgreSQL upsert layer
│   │   └── pipeline.py                # Orchestrator (CLI entry point)
│   ├── analytics/     # (extend with Python analytics scripts)
│   └── ml/
│       └── predictor.py               # Random Forest match predictor
│
├── powerbi/
│   └── DASHBOARD_SPEC.md             # Full 4-page dashboard specification
│
├── models/            # Saved ML models (auto-created)
├── logs/              # Pipeline run logs (auto-created)
├── requirements.txt
└── README.md
```

---

## Quick Start

### 1. Get an API key

Register at [football-data.org](https://www.football-data.org) — the free tier gives you access to:
Premier League, La Liga, Bundesliga, Serie A, Ligue 1, Champions League, and more.

### 2. Set up the environment

```bash
# Clone the project
git clone https://github.com/alkyramazann/Football-Analytics-BI.git
cd Football-Analytics-BI

# Create virtual environment
python -m venv .venv
source .venv/bin/activate          # Linux/Mac
.venv\Scripts\activate             # Windows

# Install dependencies
pip install -r requirements.txt
```

### 3. Configure environment variables

Create a `.env` file in the project root:

```env
FOOTBALL_API_KEY=your_api_key_here
DB_HOST=localhost
DB_PORT=5432
DB_NAME=football_analytics
DB_USER=postgres
DB_PASSWORD=your_password
```

### 4. Create the database

```bash
# Create database
psql -U postgres -c "CREATE DATABASE football_analytics;"

# Apply schema
psql -U postgres -d football_analytics -f sql/01_schema.sql
```

### 5. Run the ETL pipeline

```bash
# Fetch Premier League 2023/24 season
python -m src.etl.pipeline --competitions PL --season 2023

# Fetch multiple competitions
python -m src.etl.pipeline --competitions PL PD BL1 SA --season 2023

# Available competition codes (free tier):
# PL  = Premier League    PD  = La Liga
# BL1 = Bundesliga        SA  = Serie A
# FL1 = Ligue 1           CL  = Champions League
# PPL = Primeira Liga     ELC = Championship
```

### 6. Run SQL analytics

```bash
psql -U postgres -d football_analytics -f sql/02_analytics_queries.sql
```

### 7. Train the ML model

```python
from sqlalchemy import create_engine
from src.ml.predictor import MatchOutcomePredictor

engine = create_engine("postgresql+psycopg2://postgres:password@localhost/football_analytics")

predictor = MatchOutcomePredictor()
results = predictor.train(engine)
predictor.save()

# Predict a new match
prediction = predictor.predict({
    "home_league_pos": 3,
    "away_league_pos": 12,
    "position_diff": -9,
    "home_points": 45,
    "away_points": 28,
    "points_diff": 17,
    "home_form_pts": 10,
    "away_form_pts": 4,
    "form_diff": 6,
    "home_win_rate_hist": 0.52,
    "away_win_rate_hist": 0.28,
    "matchday": 25,
    # ... other features
})
print(prediction)
# {'prediction': 'HOME_WIN', 'probabilities': {'AWAY_WIN': 0.18, 'DRAW': 0.24, 'HOME_WIN': 0.58}}
```

### 8. Connect Power BI

Follow the instructions in `powerbi/DASHBOARD_SPEC.md` to connect Power BI Desktop
directly to PostgreSQL and build the 4-page dashboard.

---

## Database Schema

### Star Schema Design

```
                    dim_dates
                       │
dim_competitions ──────┼───── dim_teams
       │               │           │
   dim_seasons ─── fact_matches ───┘
       │
   fact_standings ─── dim_teams
```

### Key design decisions

| Decision | Rationale |
|----------|-----------|
| `fact_standings` as snapshot table | Enables point-in-time queries ("table after matchday 20"). A pure SCD2 would make this harder. |
| `dim_seasons` separate from `dim_competitions` | One competition has many seasons. Denormalizing into competitions would create update anomalies. |
| Generated columns (`outcome`, `total_goals`) | Computed at write time — avoids recalculating in every query. PostgreSQL 12+ feature. |
| `UPSERT` everywhere | Makes pipeline reruns safe. Run the pipeline daily — it only updates changed rows. |
| `dim_dates` pre-populated | Avoids JOIN on a computed expression (EXTRACT year/month) — enables index use. |

---

## SQL Analytics Queries

| Query | Technique used |
|-------|---------------|
| League table | DISTINCT ON, aggregation |
| Top scoring teams | UNION ALL, RANK() window function |
| Home vs Away analysis | Conditional aggregation |
| Rolling 5-match form | SUM() OVER with ROWS clause |
| Monthly goal trends | LAG() for month-over-month change |
| Best defensive teams | RANK(), LEFT JOIN |
| Highest scoring matches | DENSE_RANK() |
| Win/Draw/Loss distribution | FILTER clause (PostgreSQL-specific) |
| League position trajectory | LAG() for position change |
| Executive KPIs | Multi-metric single-row summary |

---

## Machine Learning

**Model:** Random Forest Classifier (scikit-learn)  
**Target:** Match outcome — HOME_WIN / DRAW / AWAY_WIN  
**Features:** 16 engineered features from league position, form, goals rates  
**Expected accuracy:** 50–56% (realistic benchmark — coin flip = 33%)

> **Why not higher?** Football is genuinely unpredictable. Studies by academic sports analysts consistently find that even the best models peak around 55-58% accuracy for three-way outcome prediction. A model claiming 75%+ is either overfitting or cherry-picking favorable data.

**Deliberate choices:**
- Chronological train/test split (not random) — avoids leaking future data
- `class_weight='balanced'` — draws are underrepresented in training data
- 5-fold cross-validation for robust accuracy estimate
- Feature importance chart saved automatically

---

## Power BI Dashboard Pages

| Page | Focus | Key visuals |
|------|-------|-------------|
| Executive Overview | Season-level KPIs | KPI cards, league table, result donut |
| Team Performance | Attacking & defensive metrics | Bar charts, scatter quadrant |
| Match Analytics | Match-level trends | Stacked bar, line, top matches table |
| Advanced Insights | Form, position trajectory | Line race, form matrix |

---

## Skills Demonstrated

| Area | Skills |
|------|--------|
| Data Engineering | REST API integration, rate limiting, retry logic, raw data persistence |
| ETL | Extract → Transform → Load, idempotent upserts, chunked bulk loading |
| Database | Star schema, normalized design, FKs, indexes, constraints, generated columns |
| SQL | CTEs, window functions, aggregations, FILTER clause, DISTINCT ON |
| Python | OOP, dataclasses, type hints, logging, argparse, dotenv |
| Machine Learning | Feature engineering, RF classifier, cross-validation, confusion matrix |
| BI / Visualization | DAX measures, Power BI data model, relationships, drill-through |

---

## Extending the Project

Some ideas for going further:

- **Add player-level data** — Football-Data.org has squad endpoints (paid tier)
- **Schedule pipeline with Apache Airflow** — daily DAG for live season tracking
- **Containerize with Docker** — `docker-compose up` for PostgreSQL + pipeline
- **Add dbt** — replace raw SQL analytics with versioned, tested dbt models
- **Deploy Power BI to web** — Power BI Service + Row-Level Security per competition
- **Add xG features** — fetch expected goals from StatsBomb open data to improve ML

---

## License

MIT — free to use for portfolio, learning, and personal projects.
Data is sourced from [football-data.org](https://www.football-data.org) under their terms of service.
