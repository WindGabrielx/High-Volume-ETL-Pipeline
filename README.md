# pipeline_test

ETL pipeline that loads sensor data from parquet files into PostgreSQL. Built with Mage.ai for orchestration and Docker for containerization.

## What it does

Reads ~43,000 parquet files (~82M rows) from `data_sample/` and loads them into a single `fact_readings` table in PostgreSQL. The whole thing runs in under 30 minutes.

## Requirements

- Docker Desktop
- Python 3.10 (for generating sample data)

## Getting started

**1. Generate the sample data**

```bash
pip install -r requirements.txt
python sampledata.py
```

This creates 43,201 parquet files in `data_sample/`. This process takes a few minutes.

**2. Start the containers**

```bash
docker-compose up -d
```

This spins up PostgreSQL on port 5432 and Mage.ai on port 6789.

**3. Create the table**

```bash
docker exec -it postgres_db psql -U admin -d pipeline_db
```

```sql
CREATE TABLE IF NOT EXISTS fact_readings (
    id              BIGSERIAL PRIMARY KEY,
    department_name VARCHAR(32),
    sensor_serial   VARCHAR(64),
    product_name    VARCHAR(16),
    product_expire  TIMESTAMP,
    create_at       TIMESTAMP
);
```

**4. Run the pipeline**

Open http://localhost:6789, go to `datawow_pipeline`, and hit Run.

The pipeline has two blocks:
- `extract_parquet` scans `data_sample/` and returns a list of file paths
- `load_postgres` reads files in batches and bulk-loads into PostgreSQL via `COPY`

Progress is logged every 5,000 files. When it finishes you should see ~82M rows in the table.

**5. Verify**

You can verify it by opening PowerShell in your project directory and running this command:

```bash
docker exec -it postgres_db psql -U admin -d pipeline_db -c "SELECT COUNT(*) FROM fact_readings;"
```

## Database

**Connection**

| | |
|---|---|
| Host | localhost |
| Port | 5432 |
| Database | xxxx |
| User | xxxx |
| Password | xxxx |

**Schema**

`fact_readings` is a flat denormalized table with no foreign keys to keep bulk inserts fast.

| Column | Type | Description |
|---|---|---|
| id | BIGSERIAL | Primary key |
| department_name | VARCHAR(32) | Department the sensor belongs to |
| sensor_serial | VARCHAR(64) | Unique sensor identifier |
| product_name | VARCHAR(16) | Product being tracked |
| product_expire | TIMESTAMP | Product expiry date |
| create_at | TIMESTAMP | Time the reading was recorded |

**Schema diagram**

![ER Diagram](ER_picture/Screenshot%202026-04-13%20124203.png)

## Design notes

The schema uses a single flat fact table with no normalization or foreign keys. The goal was bulk load speed: with 82M rows, skipping ID lookups between departments, sensors, and readings shaves significant time off the pipeline.

For the actual insert, PostgreSQL COPY (via copy_expert) replaces the usual execute_values approach. Think of it as streaming data directly into the table rather than parsing SQL row by row. Files are batched 100 at a time, concatenated into a DataFrame, then piped in as CSV. The full 82M rows loads in around 17 - 22 minutes.

Re-running the pipeline is safe because TRUNCATE fires at the start of each run so there's no risk of duplicate rows building up.

## Notes

- Mage.ai login: admin@admin.com / admin
- `data_sample/` is mounted into the Mage.ai container at `/home/src/pipeline_test/data_sample/`
- The `data_sample/` folder is gitignored - generate it locally using `sampledata.py`
