# Motor Vehicle Collisions — Advanced Data Architecture for BI

An end-to-end data engineering and business intelligence project analyzing **3.35 million motor vehicle collision records** across four U.S. cities — **New York City, Chicago, Austin, and Montgomery County (MD)**. The project covers data profiling, dimensional modeling, dual-tool ETL implementation, cloud warehousing, and interactive analytics through both Power BI and Tableau.

Built as the final project for **DAMG7370 — Designing Advanced Data Architectures for Business Intelligence** at Northeastern University.

---

## Project Overview

The goal was to unify four heterogeneous open-data sources into a single analytical warehouse capable of answering cross-city questions about crash frequency, severity, contributing factors, and temporal patterns. Each city's data exposed different attributes (NYC tracks borough and contributing factors; Austin includes micromobility; Chicago captures roadway conditions; Montgomery captures collision types and harmful events), so the modeling work required reconciling four different schemas into one consistent dimensional model.

### Data Sources

| Source | Records | Notable Attributes |
|---|---|---|
| NYC Open Data | 2.14M | Borough, ZIP, contributing factors, vehicle types |
| Chicago Data Portal | 604K | Lane count, roadway alignment, weather, lighting |
| Austin Open Data | 213K | Construction zones, micromobility incidents, injury severity |
| Montgomery Data Portal | 108K | Collision types, harmful events, traffic control, substance abuse |

---

## Architecture

The pipeline follows a layered architecture: **raw landing → staging → complete stage → dimensional warehouse → BI**.

```
┌──────────────────┐     ┌──────────────────┐     ┌──────────────────┐
│  City Open Data  │ ──▶ │   Landing Zone   │ ──▶ │   Staging Layer  │
│  (NYC / Chicago  │     │   (Parquet/CSV)  │     │  (per-city init  │
│  Austin / Mont.) │     │                  │     │  & complete tbls)│
└──────────────────┘     └──────────────────┘     └──────────────────┘
                                                            │
                                                            ▼
                          ┌──────────────────────────────────────────┐
                          │      Snowflake Dimensional Warehouse     │
                          │                                          │
                          │  FCT_ACCIDENTS (fact)                    │
                          │  ├─ DIM_DATE                             │
                          │  ├─ DIM_TIME                             │
                          │  ├─ DIM_LOCATION                         │
                          │  ├─ DIM_SOURCE                           │
                          │  ├─ DIM_VEHICLE_INVOLVED                 │
                          │  ├─ DIM_CONTRIBUTOR (SCD)                │
                          │  ├─ BRG_ACC_VEH (bridge)                 │
                          │  └─ BRG_ACC_CONTR (bridge)               │
                          └──────────────────────────────────────────┘
                                                │
                                                ▼
                          ┌──────────────────────────────────────────┐
                          │           Visualization Layer            │
                          │                                          │
                          │   Power BI (3 dashboards)                │
                          │   Tableau   (3 dashboards)               │
                          └──────────────────────────────────────────┘
```

### Dimensional Model

A **star schema** with two bridge tables to support many-to-many relationships:

- **`FCT_ACCIDENTS`** — central fact table with surrogate keys to all dimensions and metrics for injured/killed counts (motorist, pedestrian, registered, total).
- **`DIM_DATE`** / **`DIM_TIME`** — conformed date and time dimensions supporting temporal slicing (day, week, season, hour-of-day, weekday/weekend).
- **`DIM_LOCATION`** — unified geographic dimension across all four cities.
- **`DIM_SOURCE`** — source city tagging for cross-city analysis.
- **`DIM_VEHICLE_INVOLVED`** — vehicle type catalog.
- **`DIM_CONTRIBUTOR`** — contributing factors with **Type 2 SCD** to preserve historical taxonomy changes.
- **`BRG_ACC_VEH`** / **`BRG_ACC_CONTR`** — bridge tables capturing many-to-many relationships between accidents and the vehicles / contributing factors involved.

---

## Tech Stack

| Layer | Tools |
|---|---|
| **Data Profiling** | Alteryx (per-field quality assessment using 5C framework: Clean, Consistent, Comprehensive, Confirmed, Current) |
| **Ingestion & ETL** | Azure Data Factory, Talend Open Studio |
| **Transformation** | SQL, Talend `tMap` / `tNormalize`, dimensional load jobs |
| **Warehouse** | Snowflake |
| **Modeling** | Star schema with bridge tables and Type 2 SCD |
| **Visualization** | Power BI, Tableau |
| **Languages** | SQL, Python (for profiling and ad-hoc analysis) |

Both Talend and Azure Data Factory were implemented in parallel to demonstrate two industry-standard ETL tooling approaches against the same target schema.

---

## Pipeline Implementation

### 1. Data Profiling (Alteryx)
Field-level analysis across all four datasets using the 5C framework — checking for nullity, value distributions, formatting consistency, geographic validity, and recency. The profiling output drove the cleaning rules applied in the staging layer.

### 2. Initial Staging Load
Per-city `*_INIT` tables landed raw extracts from the data portals with minimal transformation — type alignment, column renames, basic format standardization.

### 3. Complete Staging Load
Per-city `*_COMP` tables applied normalization and enrichment — splitting concatenated fields, mapping contributing factors against a standardized lookup, and producing analysis-ready records.

### 4. Dimensional Load
- **Dimensions** loaded with deduplication and Type 2 SCD logic where appropriate (`DIM_CONTRIBUTOR`).
- **Fact** loaded by joining staging records against dimensions and resolving surrogate keys.
- **Bridge tables** populated to handle many-to-many relationships between accidents, vehicles, and contributing factors.

### 5. Reporting
Six dashboards total — three in Power BI, three in Tableau — covering city comparison, temporal patterns, contributing factors, and high-incident location analysis.

---

## Key Findings

A few highlights from the visualization layer:

- **NYC dominates volume** (2.14M of 3.35M total accidents, ~64%), followed by Chicago (~27%), Austin (~6%), and Montgomery (~3%).
- **Friday is the highest-incident weekday** (537K accidents); Sunday is the lowest (409K).
- **Evening hours peak** for total incidents; early morning is the lowest-volume period.
- **Fall is the highest-incident season** (~900K) — Winter is the lowest (~774K).
- **Pedestrian fatalities have steadily increased** over 2010–2024, while non-pedestrian fatalities show a slight decline post-2020.
- **Data quality gap**: 73% of records have "Undefined" or "Unable to Determine" as the contributing factor — a significant signal for improving incident reporting.

---

## Business Questions Answered

The warehouse supports SQL queries for questions including:

- Total accidents per city and per area
- Top accident-prone streets per city (top 3 / top 5)
- Injury-only vs. fatal accident counts (overall and by city)
- Pedestrian involvement rates
- Seasonality, day-of-week, and time-of-day distributions
- Motorist injury and fatality counts per city
- Top contributing factors across all accidents

Example SQL queries are included in the report.

---

## Team

This was a Group 2 project for DAMG7370 under **Prof. Naveen Kuragayala**.

- **Aamir Jawadwala** — Project lead. Owned the dimensional model design, Snowflake warehouse setup, Azure Data Factory pipelines, and the Power BI visualization layer.
- **Yash Khavnekar**
- **Niki Choksi**

---

## Repository Structure

```
Motor-Vehicle-Collisions/
├── Phase - 1/                      # Data profiling & initial schema design
│   ├── Austin_Profiling.yxmd       # Alteryx profiling workflows
│   ├── Chicago_Profiling.yxmd
│   ├── NYC_Profling.yxmd
│   ├── ADF_Data_Loading.pdf        # Azure Data Factory documentation
│   ├── FinalProject_DDL_queries.sql
│   ├── Mapping_Document.xlsx
│   └── Project Report - Phase 1.pdf
│
├── Phase - 2/                      # Full ETL, warehouse load, and BI
│   ├── Final_Project/              # Talend project files
│   ├── Final report.pdf            # Complete project report
│   └── Final_Project.DM1           # Dimensional model
│
└── README.md
```

---

## Full Project Report

For complete details — including field-by-field profiling, full DDL, ETL job documentation, and dashboard screenshots — see [`Phase - 2/Final report.pdf`](./Phase%20-%202/Final%20report.pdf).
