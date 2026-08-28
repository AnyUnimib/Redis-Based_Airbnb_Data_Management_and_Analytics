# Redis-Based Airbnb Data Management & Analytics for New York City

**Any Das**

*Originally developed as a group project (2 contributors); this repository reflects my individual exploration and contribution to the pipeline and analysis.*

An automated Big Data pipeline that ingests, cleans, and loads over **36,000 NYC Airbnb listings** (plus 13M+ calendar records and ~1M reviews) into **Redis**, then runs real-time analytical queries to surface pricing, occupancy, and amenity insights across the city's rental market.

## Why this project

Most Airbnb analysis projects stop at "load a CSV, run pandas." This one asks a different question: what changes when you treat the dataset as something that needs to be *queried in real time*, not just analyzed once? Redis was chosen deliberately over a traditional SQL setup to explore how an in-memory, NoSQL data model changes both the engineering (how you structure listings, rankings, and location data) and the kind of questions you can answer instantly versus with a batch query.

## Data

Sourced from [Inside Airbnb](http://insideairbnb.com/get-the-data/)'s New York City snapshot (Oct 1, 2025):

| Dataset | Volume | Role |
|---|---|---|
| `listings.csv.gz` | 36,111 listings, 70+ fields | Core property attributes → Redis Hashes & geospatial index |
| `calendar.csv.gz` | 13,180,517 rows | Daily price/availability → occupancy analysis & price imputation |
| `reviews.csv.gz` | 986,597 rows | Guest reviews → qualitative signal layer |

## Pipeline

```
Inside Airbnb (raw .csv.gz)
  └─ Automated download
      └─ Cleaning & preprocessing (pandas)
          └─ Redis data modeling (Hashes / Sets / Sorted Sets / Geo)
              └─ Analytical queries → business insights & visualizations
```

Everything runs end-to-end with a single **Runtime → Run all** in Colab — no manual steps.

### Data cleaning — the interesting part

Standard mean imputation doesn't work for a market as heterogeneous as NYC (a $2,573/night Midtown average next to a $105/night Bushwick average would make a global mean meaningless). Instead, a **tiered imputation strategy** was built:

1. **Price cleaning** — strip currency symbols/commas, cast to float.
2. **Primary imputation** — missing prices filled from that listing's *own* average daily price in the calendar dataset (real historical signal, not a statistical guess).
3. **Secondary imputation** — still missing? Fill with the median price for that `neighbourhood × room_type` group.
4. **Final fallback** — global median, only as a last resort.
5. Missing names → `"Unknown Listing"`; missing review counts/ratings → `0` to keep downstream aggregation safe.
6. **Feature selection** — reduced 70+ raw fields down to 10 core attributes (id, name, host_id, neighbourhood, room type, price, review count, rating, lat/long) before writing to Redis, to keep memory usage lean.

### Redis data modeling

| Structure | Used for | Why |
|---|---|---|
| **Hashes** | `listing:<id>` → name, price, rating, room type, neighbourhood | Constant-time field-level reads/updates without parsing full JSON objects |
| **Sets** | Listings grouped by neighbourhood + a global city set | Fast membership tests and intersections — e.g. "all listings in Midtown" resolves instantly |
| **Sorted Sets** | Listings ranked by rating | O(log n) top-N queries for leaderboard-style ranking, no runtime sort needed |
| **Geospatial Index** | Listing lat/long (validated before indexing) | Foundation for future radius/proximity queries |

## Key Insights

**Rating and price aren't linearly related.** The top-rated (5.0) listings span the full price spectrum — from $629/night entire homes in Upper East Side to $41/night private rooms in Bedford-Stuyvesant and Harlem. High quality doesn't require a high budget; it requires the right neighbourhood.

**Premium ≠ satisfied.** Midtown has the highest average price (**$2,573**) but one of the *lowest* average ratings (**2.58**), while Harlem, at a fraction of the price (**$155**), rates meaningfully higher (**3.55**). Commercial tourist hubs and guest satisfaction pull in different directions.

**Inventory shortages hide in residential neighbourhoods, not tourist cores.** New Dorp showed **0% availability** (effectively sold out), with Two Bridges and Inwood close behind at ~20% — a signal of underserved demand outside Manhattan's obvious hotspots, and a concrete opportunity flag for hosts.

**Guest satisfaction has a clear, findable baseline.** Filtering to listings rated 4.5+ (21,367 properties) and mining their amenities: **Smoke alarm (93.3%)** and **Wifi (92.1%)** appear in nearly all of them — not differentiators, but non-negotiable baseline expectations. Kitchen, essentials, and carbon monoxide alarms round out the top 5, with `hangers`, `hair dryer`, and `iron` showing guests consistently value practical domesticity over hotel-style luxury extras.

## Repository Structure

```
├── BIG_DATA_&_BI.ipynb           # Main pipeline: ingestion, cleaning, Redis loading, queries, viz
├── Standalone_Script.ipynb       # One-click Colab runner (installs deps, executes main notebook end-to-end)
├── Big_Data_BI_Report.pdf        # Full technical report
├── Presentation.pdf              # Slide deck summary
└── README.md
```

## How to Run

1. Upload all files to Google Colab.
2. Open `Standalone_Script.ipynb`.
3. **Runtime → Run all.**

This automatically installs dependencies, starts a local Redis server, downloads the Airbnb data, runs the full pipeline, and writes `executed_BIG_DATA_&_BI.ipynb` with all outputs populated — no manual setup required.

## What I'd improve next

- Move geospatial indexing from "extensible" to actually queried — build out real radius/proximity search (e.g. "listings within 1km of Central Park").
- Add a lightweight API layer (Flask/FastAPI) on top of Redis so the queries are callable as an actual service, not just notebook cells.
- Benchmark Redis query latency against an equivalent Postgres/SQL setup on the same dataset, to make the "why Redis" argument with numbers instead of just architecture.
- Extend the amenity analysis with a proper statistical test (not just frequency counts) to confirm which amenities *causally* correlate with higher ratings.

## Tech Stack
Python · pandas · Redis (Hashes, Sets, Sorted Sets, Geo) · Google Colab
