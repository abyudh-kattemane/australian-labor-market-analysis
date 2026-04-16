# Australian Labour Market Analysis Dashboard

An interactive Power BI dashboard analysing employment trends, industry composition, geographic labour market performance, and job seeker insights across Australia's five major cities using ABS data spanning January 2019 to December 2025.

---

## Dashboard Preview

> Screenshots to be added after export from Power BI. Suggested filenames:
> - `screenshots/00_cover.png`
> - `screenshots/01_executive_overview.png`
> - `screenshots/02_industry_deep_dive.png`
> - `screenshots/03_geographic_analysis.png`
> - `screenshots/04_job_seeker_insights.png`
> - `screenshots/05_scatter_focus.png`

---

## Dashboard Pages

| Page | Description | Key Visual |
|---|---|---|
| Executive Overview | National unemployment and employment trends Jan 2019–Dec 2025 | Line chart with COVID annotation and full-employment reference line |
| Industry Deep Dive | Employment by sector across 6 states, growth analysis since 2019 | Top 5 industry trends, employment growth diverging bar chart |
| Geographic Analysis | SA4 regional breakdown across Adelaide, Brisbane, Melbourne, Perth, Sydney | SA4 unemployment bar chart with city-level comparison |
| Job Seeker Insights | Jobs vs income scatter analysis for 19 industries across 5 cities | Bubble chart with quadrant lines — workforce size, income, and job volume |

---

## Key Insights

- Australian unemployment fell from a **7.44% COVID peak (July 2020)** to **4.10% by December 2025**
- Australia has recorded **27 consecutive months below the 4% full-employment threshold** since 2022
- **Health Care employs 2.4M Australians** — nearly double the next largest sector (Professional Services at 1.37M)
- **Adelaide unemployment (3.55%) consistently outperforms the national average (4.10%)** since 2022
- Within Adelaide, a **2.1 percentage point gap** exists between Adelaide North (4.90%) and Adelaide Central & Hills (2.78%)
- Adelaide North peaked at **10.02% unemployment during COVID** — the highest of any SA4 region in the dataset
- **Mining pays $115K median income** with only 9K Adelaide jobs vs Health Care at $39K with 190K jobs — a stark jobs-income tradeoff
- Three industries have been contracting since 2019: **Agriculture (-22.4%), Wholesale Trade (-15.5%), ICT (-11.9%)**
- Total Australian employment grew **15.5%** from 12.7M (January 2019) to 14.7M (December 2025)

---

## Technical Stack

### Data Pipeline (Python)

| Component | Detail |
|---|---|
| Language | Python 3 |
| Environment | Jupyter Notebooks |
| Core Libraries | pandas, numpy, matplotlib |
| Raw data format | ABS Excel workbooks (.xlsx) with multi-row metadata headers |

### Visualisation (Power BI)

| Component | Detail |
|---|---|
| BI Tool | Microsoft Power BI (March 2026) |
| Data Model | Star schema — DimDate, DimCity, 4 fact tables |
| DAX Measures | 22 custom measures across 4 groups: National, City, Industry, Jobs & Income |
| Calculated Columns | SA4 Display Name (MID+FIND), Industry Display Name (SWITCH), IsCovid flag |
| Relationships | 5 relationships — 1-to-many, single cross-filter direction |
| Pages | 5 pages including 1 tooltip page |

---

## Data Cleaning Pipeline

Each raw ABS dataset required a dedicated cleaning notebook to handle ABS-specific formatting — multi-row metadata headers, Series ID columns, wide monthly layouts, and suppressed cells — before loading into Power BI.

### `02_clean_national_data.ipynb` → `employment_trends_national.csv`

**Source:** ABS Labour Force Survey (Cat. 6202.0) — `6202001.xlsx`

Key steps:
- Parsed ABS multi-sheet Excel workbook, using `skiprows` to bypass metadata rows
- Built a Series ID mapping dictionary from the Index sheet to translate cryptic ABS column codes into human-readable descriptions
- Filtered for **Seasonally Adjusted** series only, selecting Employed Total, Unemployed Total, Unemployment Rate, Participation Rate, and Labour Force Total for Persons
- Converted employment figures from ABS thousands to actual persons
- Applied date filter from January 2019 onwards to capture pre-COVID baseline through recovery
- Quality checks: null counts, duplicate dates, date continuity, value range validation
- Verified output with matplotlib line charts before export

---

### `MRM1_clean_employment_sa4_adl.ipynb` → `employment_sa4_major_cities.csv`

**Source:** ABS Regional Labour Markets (MRM1.xlsx)

Key steps:
- Parsed 4 separate Excel sheets (Tables 1, 2, 5, 6) for employed persons, unemployed persons, unemployment rate, and participation rate respectively
- Used `str.extract()` with regex to pull numeric SA4 codes from text labels like `"401 Adelaide - Central and Hills"`
- Defined SA4 code ranges for all 5 major cities covering 36 regions (Sydney 115–128, Melbourne 206–213, Brisbane 301–305, Adelaide 401–404, Perth 503–507)
- Applied `pd.melt()` to reshape wide monthly columns into long format (one row per SA4 per month)
- Performed sequential merge of 4 dataframes on `date` + `sa4_code`
- Quality checks: null counts per column, all 36 regions present, date range continuity confirmed

---

### `employed_timeseries.ipynb` → `industry_employment_timeseries.csv`

**Source:** ABS Labour Force Detailed (Cat. 6291.0) — `6291005.xlsx`

Key steps:
- Loaded and concatenated 3 Excel data sheets (Data1, Data2, Data3) on a shared date column, dropping duplicate date columns from sheets 2 and 3
- Wrote a custom `parse_series_description()` function to extract `state`, `industry`, and `employment_type` from semicolon-delimited ABS column headers (e.g. `"New South Wales ; Health Care and Social Assistance ; Employed total"`)
- Filtered for 6 states (Australia + 5 major states) and Employed Total type only
- Applied `pd.melt()` to reshape from wide (one column per series) to long format
- Standardised industry names: lowercase, spaces replaced with underscores, "All Industries" renamed to "total"
- Converted figures from ABS thousands to actual persons

---

### `Jobs___Income_by_Industry.ipynb` → `jobs__income_by_industry_geography_2023.csv`

**Source:** ABS Jobs and Income by SA2 2022–23 (Table 3.2)

Key steps:
- Built a `get_sa4_from_sa3()` function using string slicing — the first 3 digits of any SA3 code equal its parent SA4 code — to filter SA3 geographies without a lookup table
- Filtered 143 metropolitan SA3 geographies across 5 cities using the SA4 code ranges defined above
- Applied `pd.melt()` separately to jobs and income tables before merging on SA3 code + industry
- Converted jobs figures from ABS thousands to actual persons using `* 1000`
- Handled suppressed ABS cells (published as blank) with `fillna(0)` — these appear as $0 income in the dataset and are excluded in Power BI via visual-level filters
- Renamed and reordered columns for direct Power BI compatibility

---

## Data Model

```
DimDate (Date) ──────────────────────────────────────────┐
     │                                                    │
     │ 1-to-many                                          │
     ├──────────────────────┬───────────────────┐        │
     ▼                      ▼                   ▼        │
NationalTrends         SA4Cities        IndustryTimeSeries│
(84 rows)              (3,024 rows)     (3,360 rows)      │
                           │                              │
                    DimCity (City)                        │
                           │ 1-to-many                    │
                           ▼                              │
                       JobsIncome ───────────────────────-┘
                       (2,860 rows)
```

---

## DAX Measures Reference

### National Measures
| Measure | Purpose |
|---|---|
| `Unemployment Rate Latest` | Latest national unemployment rate (ignores date slicer) |
| `Unemployment Rate` | Context-aware rate for chart axes |
| `Unemployment Rate YoY Change` | Year-on-year percentage point change |
| `Employed Total Latest` | Latest employed persons count |
| `Employed Millions` | Latest employed total formatted in millions |
| `Participation Rate Latest` | Latest participation rate |
| `Labour Force Latest` | Latest total labour force size |

### City / SA4 Measures
| Measure | Purpose |
|---|---|
| `City Unemployment Rate Latest` | Latest unemployment rate for selected city |
| `City Unemployment Rate` | Context-aware rate for charts |
| `City Employed Total Latest` | Latest employed total for selected city |
| `City Unemployed Total Latest` | Latest unemployed total for selected city |
| `City Participation Rate Latest` | Latest participation rate for selected city |
| `City Highlight Colour` | Conditional colour — selected city navy, others light blue |

### Industry Measures
| Measure | Purpose |
|---|---|
| `Industry Employment` | Context-aware employment sum for charts |
| `Industry Employment Latest` | Latest employment by industry (Australia, excl. total) |
| `Industry Share of Total` | Industry as % of total employment |
| `Employment Growth Pct` | Growth from first to last date in selected period |
| `Industry Bar Colour` | Conditional colour by growth rate |
| `Growth Bar Colour` | Blue for growth, red for contraction |

### Jobs & Income Measures
| Measure | Purpose |
|---|---|
| `Total Jobs Display` | Total jobs excluding total rows |
| `Avg Median Income` | Average median income excluding total and zero rows |
| `Top Paying Industry` | Text measure — highest median income industry |
| `Largest Industry by Jobs` | Text measure — highest job count industry |
| `Avg Jobs Constant` | Fixed average for scatter chart X-axis quadrant line |
| `Avg Income Constant` | Fixed average for scatter chart Y-axis quadrant line |

---

## Data Sources

All data sourced from the Australian Bureau of Statistics (ABS):

| Dataset | ABS Catalogue | Description |
|---|---|---|
| `employment_trends_national.csv` | 6202.0 | Monthly national employment, unemployment, and participation rates Jan 2019–Dec 2025 |
| `employment_sa4_major_cities.csv` | 6291.0 | Monthly SA4-level employment data across 36 regions in 5 major cities |
| `industry_employment_timeseries.csv` | 6291.0 | Monthly employment by industry and state across 20 industries and 6 states |
| `jobs__income_by_industry_geography_2023.csv` | Jobs and Income by SA2 | 2022–23 snapshot of employee jobs and median income by SA2/SA3 geography and industry |

**Data note:** Median income figures in the Jobs & Income dataset reflect per-job income including part-time and casual workers. Accommodation & Food and Retail figures appear low due to high casual employment rates and should not be interpreted as annual full-time salaries.

---

## Repository Structure

```
australian-labor-market-analysis/
├── README.md
├── dashboard/
│   └── Labour_Dashboard.pbix
├── data/
│   ├── raw/
│   │   ├── 6202001.xlsx
│   │   ├── 6291005.xlsx
│   │   ├── MRM1.xlsx
│   │   └── Table_3_Employee_jobs_income_202223.xlsx
│   └── processed/
│       ├── employment_trends_national.csv
│       ├── employment_sa4_major_cities.csv
│       ├── industry_employment_timeseries.csv
│       └── jobs__income_by_industry_geography_2023.csv
├── notebooks/
│   ├── 02_clean_national_data.ipynb
│   ├── MRM1_clean_employment_sa4_adl.ipynb
│   ├── employed_timeseries.ipynb
│   └── Jobs___Income_by_Industry.ipynb
├── screenshots/
│   ├── 00_cover.png
│   ├── 01_executive_overview.png
│   ├── 02_industry_deep_dive.png
│   ├── 03_geographic_analysis.png
│   ├── 04_job_seeker_insights.png
│   └── 05_scatter_focus.png
└── docs/
    └── Abyudh_Australian_Labour_Market_Analysis.pdf
```

---

## How to View the Dashboard

**Option 1 — Power BI Desktop (Full interactive experience):**
1. Download and install [Power BI Desktop](https://powerbi.microsoft.com/desktop) (free)
2. Clone or download this repository
3. Update the data source paths: Home → Transform data → Data source settings → update file paths to your local `/data/` folder
4. Refresh data and explore all four pages

**Option 2 — PDF (Static view):**
Download `docs/Abyudh_Australian_Labour_Market_Analysis.pdf` for a static snapshot of all pages.

**Option 3 — Screenshots:**
Browse the `/screenshots/` folder for high-resolution page previews.

---

## About

Built as a portfolio project demonstrating Power BI data modelling, DAX calculation, and dashboard design skills applied to real ABS labour market data.

**Skills demonstrated:**

*Python / Data Engineering:*
- ABS Excel workbook parsing with multi-row metadata headers using `skiprows` and Index sheet mapping
- Regex-based column extraction with `str.extract()` for structured text fields
- Custom parsing functions for semicolon-delimited ABS series descriptions
- Wide-to-long reshaping with `pd.melt()` across multiple datasets
- Multi-sheet concatenation and sequential dataframe merging
- Hierarchical geographic filtering using SA3-to-SA4 code relationships
- Data quality validation: null checks, duplicate detection, date continuity, value range assertions

*Power BI / Analytics:*
- Star schema data model design with calculated dimension tables
- Power Query transformations: type casting, null handling, column renaming
- DAX measures including time intelligence (`DATEADD`), context manipulation (`CALCULATE`, `ALL`, `ALLEXCEPT`), iterators (`TOPN`, `FIRSTNONBLANK`), and conditional formatting measures
- Multi-page dashboard design with cross-page slicer sync, bookmark navigation, and edit interactions
- Dynamic visual titles driven by DAX measures
- Report page tooltips and scatter chart quadrant analysis
- Conditional bar and column colouring via field value formatting

---

*Data sourced from the Australian Bureau of Statistics. All analysis reflects publicly available ABS data.*
