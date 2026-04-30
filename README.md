# 🪰 PESTCAST — Fruit Fly Pathway Risk Dashboard

**Dual-frontend intelligence platform for forecasting invasive fruit fly introduction risk into the contiguous United States via foreign air passenger and cargo pathways.**

[![Python](https://img.shields.io/badge/Python-3.11-3776AB?logo=python&logoColor=white)](https://www.python.org/)
[![Streamlit](https://img.shields.io/badge/Streamlit-analytics%20platform-FF4B4B?logo=streamlit&logoColor=white)](https://streamlit.io/)
[![Leaflet](https://img.shields.io/badge/Leaflet.js-geospatial%20dashboard-199900?logo=leaflet&logoColor=white)](https://leafletjs.com/)
[![scikit-learn](https://img.shields.io/badge/scikit--learn-Poisson%20GLM-F7931E?logo=scikit-learn&logoColor=white)](https://scikit-learn.org/)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

> USDA APHIS PPQ × CSU Hackathon 2026 · Prompt 1 · Six target species · Two complementary front ends · CY2026 forecasts + 90-day operational window.

---

## Why this project

> *"Which ports are most exposed right now? Where should inspectors focus their hours this season?"*

Every year, exotic fruit flies arrive at U.S. ports of entry aboard international passengers and fresh commodity shipments. USDA APHIS PPQ intercepts hundreds of specimens annually — but interception data alone lags the true introduction pressure by weeks to months.

PESTCAST is our team's answer: an end-to-end **pathway-first** pipeline that quantifies introduction risk directly from BTS air traffic, USDA commodity imports, EPPO pest presence records, and climate suitability, producing **operational 90-day risk forecasts** for all six APHIS priority fruit fly species — with a matched geospatial dashboard for presentations and a full analytical platform for resource allocation.

| Metric | PESTCAST CY2026 |
|---|---|
| **Model** | Poisson GLM, Pseudo-R² = 0.68 |
| **Validation events** | 52 APHIS detection records (CY2025, held out) |
| **Species covered** | 6 (all APHIS PPQ priority fruit fly species) |
| **Spatial resolution** | County-level establishment risk + port-of-entry pathway risk |
| **Forecast horizon** | 90-day operational window + full CY2026 outlook |
| **Validated corridors** | Mexico → TX/CA (*A. ludens*); Thailand/Kenya → West Coast (*B. dorsalis*, *C. capitata*) — consistent with APHIS 2025–2026 bulletins and IPPC phytosanitary reports |

---

## Six target species

| Species | Common Name |
|---|---|
| *Ceratitis capitata* | Mediterranean Fruit Fly (Medfly) |
| *Bactrocera dorsalis* | Oriental Fruit Fly |
| *Anastrepha ludens* | Mexican Fruit Fly |
| *Anastrepha suspensa* | Caribbean Fruit Fly |
| *Bactrocera zonata* | Peach Fruit Fly |
| *Rhagoletis cerasi* | European Cherry Fruit Fly |

---

## Two front ends

| | **Leaflet Dashboard** `app/index.html` | **PESTCAST** `app/app.py` |
|---|---|---|
| **Stack** | Static HTML + Leaflet.js + Chart.js | Streamlit + Plotly + scikit-learn |
| **Data** | Pre-built JS bundle (`app_data.js`) | Live Parquet files in `data/processed/` |
| **Species** | 3 (Medfly, Oriental FF, Mexican FF) | All 6 target species |
| **Risk model** | Deterministic composite index | Poisson GLM (Pseudo-R² = 0.68) |
| **Forecasting** | Monthly index, current year | CY2026 forecast + 90-day operational window |
| **Best for** | Quick geospatial overview, presentations | Operational analysis, resource allocation |

---

## How it works

At its core, PESTCAST computes a **pathway-weighted introduction risk** for each `(origin_country × dest_port × species × month)` cell, then layers a calibrated Poisson GLM to produce detection-probability forecasts and county-level establishment risk maps.

```
BTS T-100 (2015–2025) ──┐
USDA FAS GATS imports ──┼──→ 01_acquire_data.py ──→ 02_build_join_table.py ──→ risk_table.parquet
EPPO pest presence ──────┤
APHIS interceptions ─────┘
        │
        ├──→ 03_fit_risk_model.py    ──→ predictions.parquet          (Poisson GLM)
        ├──→ 04_marginal_value.py    ──→ marginal_value.parquet       (inspector-hour optimizer)
        ├──→ 05_network_features.py  ──→ network features             (airport-county graph)
        ├──→ 05_build_app_data.py    ──→ app/data/app_data.js         (Leaflet bundle)
        └──→ 06_climate_suitability.py ──→ climate_suitability_by_county.parquet
                                     │
WorldClim tavg/prec ──┐   07_county_predict.py ──→ county_predictions.parquet
USDA CDL (2025) ──────┘   08_backtest.py / 09_surveillance_backtest.py
                                     ▼                         ▼
                            app/index.html                app/app.py
                      (Leaflet choropleth +         (PESTCAST Streamlit —
                       flight arcs + ports)          6-tab analytics platform)
```

**Risk score formula:**

```
risk = (passengers + freight / 5,000) × pest_presence_score
```

Aggregated monthly per `(origin_country × dest_port × species)`.

Pest presence scored from EPPO distribution records: `3` = widespread · `2` = restricted · `1` = few occurrences / transient.

### Why we built it this way

- **Two complementary tools, not one.** The Leaflet dashboard is a zero-dependency presentation layer — shareable without a Python environment. The Streamlit app is the full analytical platform: Poisson GLM forecasts, marginal surveillance value, county-level establishment risk, and exportable leadership briefings.
- **Pathway-weighted, not model-predicted (Leaflet).** The static dashboard computes a deterministic composite index — volume × pest_score — producing interpretable, auditable scores. The Streamlit app layers a fitted Poisson GLM for detection-calibrated forecasts.
- **Cargo scaling.** T-100 FREIGHT is in pounds; passengers are integer headcounts. Raw freight values are ~5,000× larger. The `freight / 5,000` factor approximately equalises their contributions. The *Include Cargo Risk* toggle lets analysts compare passenger-only vs. combined pathway risk.
- **Pre-computation at build time.** `05_build_app_data.py` pre-aggregates all risk by `(species × month × country)` and `(species × month × US_port)`. The browser performs no heavy aggregation at runtime — all slider and filter updates are O(n_countries) lookups.
- **90-day operational window.** The PESTCAST app flags the next ~90 days as the operationally reliable forecast horizon. Month cells outside this window are rendered faded and labelled "outlook" to distinguish near-term actionable risk from longer-range projections.

---

## Dashboard features

### Leaflet Dashboard (`index.html`)

**Map layers** (all toggleable):
- Country choropleth in two modes: *Pest Presence* (EPPO score categories) or *Pathway Risk* (gradient heatmap — flight volume × pest score). Choropleth opacity encodes pathway volume so countries with the same EPPO score are visually differentiated by traffic intensity.
- Great-circle flight arcs for the top 15 / 30 / 50 risk routes, coloured and weighted by risk tier.
- U.S. port-of-entry bubbles sized by total inbound risk score; click for full origin breakdown.

**Controls:** Month slider (Jan–Dec) with play/pause animation · Species filter · Cargo toggle · Port IATA labels · Focus CONUS / World view

**Sidebar panels** (each collapsible): Risk summary stats · Top 12 risk pathways with species badges · U.S. port exposure chart · Monthly passenger flow trend · Host commodity imports by partner country · Recent detections & IPPC alerts feed · Risk model methodology callout

---

### PESTCAST Streamlit App (`app/app.py`) — 6 tabs

**Sidebar:** Species focus pill (all 6) · Month selector · Cargo toggle · Data vintage stamp

| Tab | Content |
|---|---|
| **Priorities** | Combined pathway × climate risk map (county-level bubbles over climate suitability choropleth) · Top 10 high-risk counties with operational status badges (High / Medium / Watch) · 90-day aggregate view · Annual county trend with 5-year sparklines · County seasonality heatmap · County driver breakdown by origin country · Multi-species hotspot table (counties ranking top-20 for ≥2 species simultaneously) · Printable HTML leadership briefing export |
| **Surveillance** | Inspector-hour allocation optimizer with marginal-value model and diminishing-returns curve · Hour deployment slider (200–20,000 hrs) · 90-day operational focus + annual budget outlook |
| **Pathways** | Top origin → U.S. port pathways filtered to EPPO-established countries · Monthly top origin countries and ports of entry · Annual exposure share by all 6 species |
| **Establishment** | County-level climate suitability — species-specific WorldClim tavg/prec envelope · County choropleth of favorable establishment conditions · Pathway risk vs. climate suitability scatter (identifies high-arrival + high-establishment counties) |
| **Model** | Poisson GLM diagnostics — Pseudo-R² summary · Largest under-prediction callout · Coefficient table and residual plot · State-level network structure visualization |
| **About** | Data sources · methodology · data vintage · version info |

---

## Data sources

| Source | Use | Years | Access |
|---|---|---|---|
| [BTS T-100 International Segment](https://www.transtats.bts.gov/) | Passenger + freight volumes by route | 2015–2025 | BTS TranStats download |
| [USDA FAS GATS](https://apps.fas.usda.gov/gats/) | Host commodity imports (kg) by partner country | 2015–2026 | GATS Standard Query |
| [EPPO Global Database](https://gd.eppo.int/) | Pest presence status per country, 6 target species | current | gd.eppo.int distribution export |
| [APHIS PPQ Program Data](https://www.aphis.usda.gov/) | Interception / detection validation records | 2015–2026 | APHIS public bulletins |
| [IPPC Pest Reports](https://www.ippc.int/) | Phytosanitary event feed | 2025–2026 | IPPC REST API |
| [OurAirports](https://ourairports.com/) | Airport metadata — IATA, lat/lon | current | davidmegginson.github.io |
| [Natural Earth 50m](https://www.naturalearthdata.com/) | Country boundaries GeoJSON | current | nvkelso/natural-earth-vector |
| [WorldClim v2.1](https://www.worldclim.org/) | Monthly avg temperature + precipitation rasters | 1970–2000 normals | worldclim.org |
| [USDA NASS CDL](https://nassgeodata.gmu.edu/CropScape/) | U.S. host crop acreage mask | 2025 | CropScape / GEE |
| [ISO 3166-1 country codes](https://github.com/datasets/country-codes) | Country name ↔ ISO2 reconciliation | current | datasets/country-codes |

> No API keys are required for either dashboard. T-100 and GATS data require manual export from BTS TranStats and USDA FAS GATS portals — see [`DATA_PLAN.md`](DATA_PLAN.md) for step-by-step instructions.

---

## Quick start

### Prerequisites

- Python 3.11+
- T-100 and GATS CSV exports (see `DATA_PLAN.md` for step-by-step instructions)

### Setup

```bash
git clone https://github.com/bqhorsfall/PESTCAST.git
cd PESTCAST
python3 -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
```

### Run the Leaflet dashboard (no pipeline required if `app_data.js` is present)

```bash
cd app && python3 -m http.server 8765
# open http://localhost:8765
```

### Run the PESTCAST Streamlit platform

```bash
.venv/bin/streamlit run app/app.py
# opens automatically at http://localhost:8501
```

Requires processed Parquet files in `data/processed/` — generated by the pipeline below.

---

## Reproducing the full pipeline

```bash
# 1. Check which raw data files are present
python scripts/01_acquire_data.py --check

# 2. Auto-download files that can be fetched directly (airports, country codes, GeoJSON)
python scripts/01_acquire_data.py --auto

# 3. Build the join table from all raw CSVs
python scripts/02_build_join_table.py

# 4. Fit the Poisson GLM risk model
python scripts/03_fit_risk_model.py

# 5. Compute marginal surveillance value per county
python scripts/04_marginal_value.py

# 6. Compute climate suitability by county (WorldClim + CDL)
python scripts/06_climate_suitability.py

# 7. Generate county-level risk predictions
python scripts/07_county_predict.py

# 8. Build the Leaflet app data bundle
python scripts/05_build_app_data.py
```

`05_build_app_data.py` is idempotent — re-running regenerates the bundle from whatever raw files are present. Expected runtime: ~15 seconds for the full dataset.

---

## Project layout

```
scripts/
  01_acquire_data.py           auto-download + manual instructions for every source
  02_build_join_table.py       merge T-100, GATS, EPPO, APHIS → risk_table.parquet
  03_fit_risk_model.py         Poisson GLM on APHIS detection records
  04_marginal_value.py         marginal surveillance value per county
  05_build_app_data.py         process raw CSVs → app/data/app_data.js (Leaflet bundle)
  05_network_features.py       airport-county network graph features
  06_climate_suitability.py    WorldClim + CDL → climate_suitability_by_county.parquet
  07_county_predict.py         county-level CY2026 risk predictions
  08_backtest.py               model backtesting
  09_surveillance_backtest.py  surveillance allocation backtesting

data/
  raw/                         one file per source — gitignored
    t100_international_*.csv   BTS T-100, 2015–2025
    eppo_*.csv                 EPPO distribution records, 6 species
    gats_host_imports.csv
    aphis_validation.csv
    ippc_fruit_fly_pest_reports_2025_2026.csv
    airports.csv
    countries.geojson
    wc2.1_10m_tavg_*.tif       WorldClim temperature normals (12 months)
    wc2.1_10m_prec_*.tif       WorldClim precipitation normals (12 months)
    cdl_2025_clipped.tif       USDA CDL host crop mask
  processed/                   generated by pipeline — consumed by app/app.py
    risk_table.parquet
    predictions.parquet
    marginal_value.parquet
    climate_suitability_by_county.parquet
    county_predictions.parquet

app/
  index.html                   single-file Leaflet.js + Chart.js dashboard (no server needed)
  app.py                       PESTCAST Streamlit analytics platform
  data/
    app_data.js                generated JS bundle (pest presence + routes + GeoJSON)

DATA_PLAN.md                   full data acquisition plan and source rationale
```

---

## Limitations & honest caveats

- **Poisson GLM, not a deep model.** Pseudo-R² of 0.68 on 52 detection events is solid for this data volume, but the model has limited capacity to capture non-linear interactions between pathway pressure and establishment suitability. A richer model would require substantially more labeled interception records.
- **Deterministic risk index (Leaflet).** The static dashboard's composite score — volume × pest_score — is interpretable and auditable, but it is not calibrated against detection frequency. Two countries with the same score can have meaningfully different real-world introduction probabilities.
- **EPPO presence scores are coarse.** The three-tier presence score (widespread / restricted / transient) collapses substantial within-country heterogeneity. Sub-national pest distribution data was not available at the required geographic resolution for all six species.
- **CDL 2024 used as the 2025 host crop mask.** The official 2025 CDL is released after harvest; year-over-year host pixel agreement is ~85% for the relevant crops.
- **T-100 and GATS data require manual export.** These datasets cannot be auto-fetched programmatically due to portal limitations. The pipeline degrades gracefully if either is absent, but risk score coverage will be reduced.
- **Uncertainty is not quantified.** Unlike a Bayesian or ensemble approach, the Poisson GLM produces point forecasts without credible intervals. The 90-day operational window label reflects temporal reliability, not probabilistic confidence.

---

## Team

Built for the **USDA APHIS PPQ × CSU Hackathon 2026, Prompt 1**.

- **Alex Woods**

- **Blaise Horsfall**

- **Kian Jiang**

- **Hayley Smith**

---

## Acknowledgements

- **USDA APHIS PPQ** for framing the challenge and making interception and detection records publicly available.
- **Bureau of Transportation Statistics** for the T-100 International Segment database — 11 years of route-level air traffic is what makes the pathway model possible.
- **EPPO** for maintaining the Global Database of pest distribution records across all member and non-member countries.
- **IPPC** for the REST API providing near-real-time phytosanitary event feeds.
- **NASA / USGS** for the WorldClim and CDL datasets used in the climate suitability layer.
- **Colorado State University** for hosting the 2026 Hackathon.

## License

[MIT](LICENSE) — fork it, adapt it, improve it.
