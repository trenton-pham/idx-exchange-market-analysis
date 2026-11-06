# CRMLS Market Analysis @ IDX Exchange

This project processes monthly California Regional Multiple Listing Service (CRMLS) listing and closed-sale exports for market analysis, brokerage comparisons, Tableau dashboards, and exploratory sale-price modeling.

CRMLS records and derived datasets are confidential. `data/`, `docs/`, and local environment files are ignored by Git.

## Workflow

The pipeline extracts monthly data from the CoreLogic Trestle Web API, combines residential records, filters columns by missingness, adds mortgage rates, cleans records, engineers features, and filters sold-price outliers.

```text
data/raw
  → data/processed
  → data/filtered
  → data/mortgage
  → data/cleaned
  → data/feature_engineer
  → data/post_outlier
  → Tableau and modeling notebooks
```

## Project Structure

```text
idx/
├── data/                         # Local inputs, reference files, and outputs
│   ├── raw/
│   ├── processed/
│   ├── filtered/
│   ├── mortgage/
│   ├── cleaned/
│   ├── feature_engineer/
│   ├── post_outlier/
│   ├── city_boundaries/
│   └── zip/
├── notebooks/
│   ├── clean.ipynb               # Cleaning exploration and row counts
│   ├── feature_engineer.ipynb    # Feature distributions and grouped summaries
│   ├── outlier.ipynb             # Price outliers and ratio boundaries
│   └── modeling.ipynb            # Sale-price modeling
├── scripts/
│   ├── __init__.py
│   ├── extract_crmls.py
│   ├── process.py
│   ├── validation.py
│   ├── mortgage_fetch.py
│   ├── clean.py
│   ├── feature_engineer.py
│   ├── outlier.py
│   └── pipeline.py
├── tests/
│   ├── test_clean.py
│   ├── test_extract_crmls.py
│   ├── test_pipeline.py
│   └── test_process.py
└── requirements.txt
```

## Setup

The project environment uses Python 3.13. Create a virtual environment and install the pinned dependencies:

```bash
python -m venv .venv
source .venv/bin/activate
python -m pip install -r requirements.txt
```

Provide these reference files before running the full pipeline:

| Local file | Required content | Source |
| --- | --- | --- |
| `data/zip/california_valid_zip_codes.csv` | A single column named `ZIP_CODE` | [California ZIP-code boundaries](https://lab.data.ca.gov/dataset/ca-zip-code-boundaries) |
| `data/city_boundaries/City_and_County_Boundaries.geojson` | Polygon geometry and a `COUNTY_NAME` field | [City and county boundaries](https://www.californianature.ca.gov/datasets/CAnature::city-and-county-boundaries/about) |

For the ZIP reference, retain the ZIP-code column, rename it to `ZIP_CODE`, and save it at the path above. Save the boundary download at the GeoJSON path above.

## Fetching Monthly Data

The extractor obtains a short-lived Trestle access token from the team authentication proxy. Create a local `.env` file with the team-provided HTTPS endpoints:

```dotenv
CORELOGIC_API_URL=https://replace-with-team-api-url
AUTH_ENDPOINT=https://replace-with-team-auth-endpoint
```

Shell environment values override `.env`. Keep endpoint values, tokens, and MLS records out of source files, notebooks, command-line arguments, screenshots, and logs.

Fetch an inclusive month range:

```bash
python scripts/extract_crmls.py --start-month 2026-08 --end-month 2026-08
```

Ranges can cross calendar years. Each month produces:

```text
data/raw/CRMLSListingYYYYMM.csv
data/raw/CRMLSSoldYYYYMM.csv
```

Existing files are skipped so interrupted backfills can resume. Use `--force` to refresh them. Forced exports replace the previous CSV only after the new download passes validation.

The extractor checks metadata, pagination, record counts, listing-key uniqueness, date boundaries, and closed-sale status before saving an export.

## Running the Pipeline

Process the monthly CSVs in `data/raw/`:

```bash
python scripts/pipeline.py
```

Or fetch a month range before running the transformations:

```bash
python scripts/pipeline.py --fetch-start 2026-08 --fetch-end 2026-08
```

Both fetch dates are required, and the end month must be on or after the start month. Add `--force-fetch` to refresh existing exports.

The pipeline discovers all available months without a hard-coded end date. Every discovered month must have both a listing and sold file. If a pair is incomplete, processing stops with an error. When both `CRMLSSoldYYYYMM.csv` and `CRMLSSoldYYYYMM_filled.csv` exist, processing uses the `_filled` file.

The legacy `--with-reports` option references `scripts/distribution.py`, which is absent from this checkout. Omit that flag unless the script is restored.

## Transformation Stages

| Stage | Script | Processing | Output |
| --- | --- | --- | --- |
| Process | `scripts/process.py` | Combines monthly pairs and keeps `PropertyType == "Residential"`. | `data/processed/CRMLSListing.csv`, `data/processed/CRMLSSold.csv` |
| Validate | `scripts/validation.py` | Profiles missingness and key sold distributions; removes columns with more than 90% missing values. | `data/filtered/CRMLSListing_filtered.csv`, `data/filtered/CRMLSSold_filtered.csv` |
| Enrich | `scripts/mortgage_fetch.py` | Averages weekly FRED `MORTGAGE30US` observations by month and joins them by listing or close month. | `data/mortgage/*_with_mortgage.csv` |
| Clean | `scripts/clean.py` | Parses dates, normalizes names, filters California state and ZIP values, flags timeline and numeric issues, checks coordinates against county polygons, removes redundant fields, and applies known corrections. | `data/cleaned/CRMLSListing_cleaned.csv`, `data/cleaned/CRMLSSold_cleaned.csv` |
| Engineer | `scripts/feature_engineer.py` | Adds price ratios, price per square foot, close year/month, year-month labels, and listing-to-contract and contract-to-close durations. | `data/feature_engineer/CRMLSSold_feature_engineered.csv` |
| Filter | `scripts/outlier.py` | Applies a three-IQR `ClosePrice` boundary and keeps close-to-original-list-price ratios from 0.75 through 1.50. | `data/post_outlier/CRMLSSold_cleaned_out.csv` |

Cleaning flags include invalid transaction timelines, negative values, small living areas, coordinates outside California, and coordinates that do not match the reported county.

## Analysis Notebooks

- `notebooks/clean.ipynb` explores cleaning results and row counts.
- `notebooks/feature_engineer.ipynb` explores distributions and summarizes market measures by property subtype, county, MLS area, listing office, and buyer office.
- `notebooks/outlier.ipynb` examines extreme prices, source-data corrections, and alternative price-ratio boundaries.
- `notebooks/modeling.ipynb` compares linear regression, random forest, and XGBoost using an 80/20 train/test split, five-fold cross-validation, one-hot encoding, and a log-transformed target.

To reduce target leakage, the modeling notebook replaces the close-month mortgage join with the most recent available weekly FRED rate strictly before the listing-contract date.

## Tests

```bash
python -m unittest discover -s tests -v
```

The suite covers extraction retries and token refresh, pagination, schema and record validation, resumable exports, atomic file replacement, month-pair discovery, residential filtering, pipeline arguments, and county-coordinate matching.

## Dashboards and Report

### Market Analysis

![Market Analysis Dashboard](images/MarketDashboard.png)

[View the market analysis dashboard](https://public.tableau.com/views/CaliforniaRealEstateMarketAnalysis_17882190871160/MarketAnalysis?:language=en-US&:sid=&:redirect=auth&:display_count=n&:origin=viz_share_link)

### Competitive Analysis

![Competitive Analysis Dashboard](images/CompetitiveDashboard.png)

[View the competitive analysis dashboard](https://public.tableau.com/views/CaliforniaRealEstateCompetitiveAnalysis_17882211560250/CompetitiveAnalysisDashboard?:language=en-US&:sid=&:redirect=auth&:display_count=n&:origin=viz_share_link)

### Santa Clara County Market Report

![Santa Clara County Market Report](images/SantaClaraCountyMarketReport.png)

[Download the report (PDF)](SantaClaraCountyMarketReport.pdf)
