# FHIR Clinical Data Pipeline

A production-style ETL pipeline that ingests synthetic patient data from a FHIR R4 API, normalizes nested clinical resources into a relational schema, and loads it into PostgreSQL for downstream analytics.

## Overview

Healthcare interoperability is built on FHIR (Fast Healthcare Interoperability Resources), a REST/JSON standard mandated by the 21st Century Cures Act for EHR data exchange. This project demonstrates an end-to-end data engineering workflow against a real FHIR server: paginated extraction, schema normalization across 5 resource types, idempotent loading, and an analytics layer surfacing clinical KPIs.

Built as a portfolio project to demonstrate combined Healthcare IT and data engineering skills.

## Architecture

```
+----------------+       +-----------------+       +------------------+
|  HAPI FHIR R4  |  -->  |  Python ETL     |  -->  |  PostgreSQL      |
|  Public Server |       |  (extract,      |       |  (normalized     |
|                |       |   transform,    |       |   relational     |
|  Bundle JSON   |       |   load)         |       |   schema)        |
+----------------+       +-----------------+       +------------------+
                                                            |
                                                            v
                                                    +------------------+
                                                    |  Analytics Layer |
                                                    |  (SQL queries)   |
                                                    +------------------+
```

Data flows in four stages: raw FHIR Bundles are pulled via paginated REST calls, persisted to disk for replay, parsed into typed pandas DataFrames, then upserted into Postgres on natural keys.

## Tech Stack

| Layer            | Tool                          |
|------------------|-------------------------------|
| Language         | Python 3.11                   |
| HTTP client      | requests                      |
| Transformation   | pandas                        |
| Database driver  | psycopg2-binary, SQLAlchemy   |
| Config           | python-dotenv                 |
| Testing          | pytest                        |
| Database         | PostgreSQL 15                 |

## Data Source

Public HAPI FHIR R4 test server: `https://hapi.fhir.org/baseR4`

No authentication required. All data is synthetic. The pipeline ingests 5 resource types:

| Resource           | Clinical Meaning                          |
|--------------------|-------------------------------------------|
| Patient            | Demographics, identifiers, address        |
| Encounter          | Visits, admissions, episodes of care      |
| Condition          | Diagnoses with ICD-10 / SNOMED codes      |
| Observation        | Vitals and lab results with LOINC codes   |
| MedicationRequest  | Prescriptions and medication orders       |

## Database Schema

The schema is normalized to 3NF with foreign keys enforcing referential integrity. Every table carries an `ingested_at` timestamp for lineage tracking.

```
patients (patient_id PK)
    |
    +-- encounters (encounter_id PK, patient_id FK)
    |       |
    |       +-- conditions (condition_id PK, encounter_id FK)
    |       +-- observations (observation_id PK, encounter_id FK)
    |
    +-- medication_requests (request_id PK, patient_id FK)
```

Full DDL is in `src/schema.sql`. Key design decisions:

- Natural keys from FHIR (`Patient.id`, `Encounter.id`) are used as primary keys to support idempotent upserts.
- Coded values are stored alongside their code system (`code`, `code_system`, `display_text`) rather than flattened, preserving terminology context.
- Numeric and string observation values are split into separate columns (`value_numeric`, `value_string`, `value_unit`) since FHIR `Observation.value[x]` is polymorphic.

## Project Structure

```
fhir-clinical-pipeline/
├── README.md
├── requirements.txt
├── .env.example
├── docker-compose.yml          # local Postgres
├── src/
│   ├── config.py               # env vars, API base URL, batch sizes
│   ├── extract.py              # paginated FHIR Bundle fetcher
│   ├── transform.py            # resource parsers, JSON flatteners
│   ├── load.py                 # bulk upserts into Postgres
│   ├── schema.sql              # DDL
│   └── pipeline.py             # main orchestrator
├── analytics/
│   └── queries.sql             # 10 analytical queries
├── tests/
│   ├── test_extract.py
│   ├── test_transform.py
│   └── fixtures/               # sample FHIR JSON for unit tests
└── data/
    └── raw/                    # cached FHIR Bundles (gitignored)
```

## Pipeline Stages

### 1. Extract (`src/extract.py`)

Pulls FHIR resources via paginated GET requests. The HAPI server returns `Bundle` resources containing up to 20 entries per page, with a `link.relation = "next"` URL for pagination. Bundles are written to `data/raw/{resource_type}/{timestamp}.json` to decouple extraction from transformation and enable replay.

### 2. Transform (`src/transform.py`)

Each resource type has a dedicated parser that flattens FHIR's nested structure into tabular rows. Handles the common FHIR edge cases:

- Multi-valued fields (`name[].given[]`, `address[]`)
- Polymorphic value types (`valueQuantity` vs `valueString` vs `valueCodeableConcept`)
- Missing optional fields (defaulted to NULL, not dropped)
- Code system disambiguation (LOINC vs SNOMED vs ICD-10)

Output is a dict of pandas DataFrames keyed by table name.

### 3. Load (`src/load.py`)

Bulk upserts via `psycopg2.extras.execute_values` with `ON CONFLICT (id) DO UPDATE`. Loads are wrapped in a single transaction per resource type. Row counts and elapsed time are logged at INFO level.

### 4. Analytics (`analytics/queries.sql`)

10 queries demonstrating clinical insight:

1. Patient demographic distribution by age band and gender
2. Top 20 diagnoses by frequency (ICD-10)
3. Average vital signs (BP, heart rate, BMI) by age group
4. Encounter volume by month and class (inpatient, outpatient, emergency)
5. Most prescribed medications and prescriber patterns
6. Patients with multiple chronic conditions (comorbidity analysis)
7. Lab result outliers flagged against reference ranges
8. Readmission rate within 30 days
9. Average length of stay by encounter class
10. Data quality summary (null rates, row counts, ingest latency)

## Setup

```bash
# 1. Clone and install
git clone https://github.com/<user>/fhir-clinical-pipeline.git
cd fhir-clinical-pipeline
python -m venv venv && source venv/bin/activate
pip install -r requirements.txt

# 2. Start Postgres
docker compose up -d

# 3. Configure
cp .env.example .env
# edit .env if needed

# 4. Initialize schema
psql $DATABASE_URL -f src/schema.sql

# 5. Run pipeline
python -m src.pipeline --resources patient,encounter,condition,observation,medication --limit 10000

# 6. Query results
psql $DATABASE_URL -f analytics/queries.sql
```

## Engineering Practices

The pipeline follows several conventions that distinguish production code from notebook scripts:

- **Idempotent**: re-running the pipeline does not duplicate rows. Upserts key on FHIR resource IDs.
- **Separation of concerns**: extract, transform, and load are independent modules with no cross-imports of business logic.
- **Configuration over hardcoding**: API URL, batch size, DB connection, and resource list are all env-driven.
- **Structured logging**: every stage emits record counts, batch numbers, and elapsed time at INFO level.
- **Tested**: pytest covers transform parsers using fixture JSON, since transformation contains the bulk of the business logic.
- **Replayable**: raw Bundles are persisted to disk so transform/load can be re-run without re-hitting the API.

## Testing

```bash
pytest tests/ -v
```

Test coverage focuses on `transform.py` parsers, validating edge cases like missing fields, polymorphic values, and multi-valued name arrays against fixture JSON captured from the live API.

## Roadmap

Items deferred to keep scope realistic, but worth noting in interviews:

- Incremental extraction using FHIR `_lastUpdated` search parameter
- Airflow or Prefect orchestration with daily DAG runs
- Great Expectations data quality suite
- dbt models for the analytics layer
- Move analytics to a columnar warehouse (DuckDB or BigQuery)
- HIPAA compliance hardening: audit logging, row-level security, encryption at rest

## Healthcare Context

A few notes for reviewers less familiar with the domain:

- **FHIR R4** is the current production standard. R5 exists but adoption is limited.
- **LOINC** codes identify lab tests and clinical observations (for example, `8480-6` = systolic blood pressure).
- **SNOMED CT** codes identify clinical concepts including diagnoses, procedures, and findings.
- **ICD-10** is the billing diagnosis code system used in the US for reimbursement.
- **PHI** (Protected Health Information) handling is governed by HIPAA. This project uses only synthetic data.

## License

MIT
