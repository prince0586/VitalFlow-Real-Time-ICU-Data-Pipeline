# VitalFlow-Real-Time-ICU-Data-Pipeline

1. Project Overview

VitalFlow is a robust, modular Data Engineering pipeline designed for the healthcare sector. It simulates a high-velocity IoT environment (Intensive Care Unit) where patient vitals are continuously monitored.

The system is designed to ingest raw sensor data, enforce strict data quality contracts, quarantine erroneous records, and aggregate clean data into a "Gold" layer suitable for Business Intelligence (BI) and analytics.

Key Features:

Resilience: Implements custom retry logic with exponential backoff for database stability.

Data Quality: Automatically segregates data into "Silver" (Valid) and "Quarantine" (Invalid) tables.

Observability: Comprehensive logging and operation timers to track pipeline performance.

Incremental Loading: Uses watermark logic to fetch only new data since the last successful run.

2. System Requirements

Python Version: Python 3.8 or higher.

Operating System: Cross-platform (Windows, macOS, Linux).

Storage: Write access to the local file system (for the SQLite database file).

Environment Variables (Optional)
The system uses safe defaults, but can be configured via environment variables:

VF_DB_CONN: Database connection string (Default: sqlite:///icu_warehouse.db)

VF_BATCH_SIZE: Number of records to simulate per run (Default: 100)

VF_LOG_LEVEL: Logging verbosity (Default: INFO)

3. Libraries & Dependencies

To run this pipeline, you must install the following external libraries. Standard libraries (like os, time, logging, sqlite3) are included with Python.

requirements.txt

pandas>=1.5.0
numpy>=1.20.0
sqlalchemy>=1.4.0

4. Workflow & Architecture

The pipeline follows a standard ELT (Extract, Load, Transform) pattern with an embedded Quality Gate.

Step 1: Initialization
The system initializes the VitalFlowPipeline class, setting up the DataWarehouse connection and the IoTGenerator. It checks the database for a "watermark" (the timestamp of the last successfully loaded record) to ensure no data duplication.

Step 2: Extraction (Simulation)
The IoTGenerator simulates an external message queue (like Kafka).

Logic: It generates synthetic Heart Rate and Blood Pressure data for 9 patients.

Noise Injection: To mimic real-world sensors, it intentionally injects null values (missing BP) or impossible values (negative Heart Rates) randomly.

Step 3: Validation (The Quality Gate)
The QualityGate class intercepts the raw dataframe before loading.

Rules:

systolic_bp must not be Null.

heart_rate must be between 0 and 300.

Routing:

Pass: Data is marked as "Silver" quality.

Fail: Data is stamped with an error_reason and routed to the "Quarantine" table.

Step 4: Loading (Persistence)
The DataWarehouse loads the split dataframes into SQLite.

Reliability: The load_batch method is wrapped in a @retry_with_backoff decorator. If the database is locked or busy, it waits and retries up to 3 times before failing.

Step 5: Aggregation (Gold Layer)
Once raw data is secured, the pipeline runs an internal SQL transformation to create the Gold Layer:

Aggregates data by Patient, Date, and Hour.

Calculates AVG and MAX heart rates.

This table (gold_patient_hourly) is optimized for dashboarding tools (e.g., Tableau, PowerBI, Streamlit).

5. Code Structure

IoTGeneratorSimulates the external data source (Sensors). Handles cold starts vs. incremental fetches.QualityGateActs as the firewall for data quality.

Returns two dataframes: clean_df and error_df.DataWarehouseManages SQL connections, executes loads, and handles ELT transformations.

Includes retry logic.VitalFlowPipelineThe orchestrator. Ties all classes together and manages the flow of execution.

6. Future Work & Roadmap

To promote this project from a local prototype to a production-grade enterprise system, the following improvements are recommended:

Database Migration:

Replace SQLite with a scalable warehouse like PostgreSQL, Snowflake, or Google BigQuery.

Why? SQLite locks the file during writes, preventing high-concurrency ingestion.

Orchestration:

Move the while True loop to a dedicated orchestrator like Apache Airflow, Prefect, or Dagster.

Why? Better handling of dependencies, backfills, and alerting on failure.

Visualization Layer:

Build a frontend using Streamlit or Grafana.

Goal: Visualize the gold_patient_hourly table to show real-time ICU trends and alert doctors of anomalies.

Containerization:

Wrap the application in Docker.

Goal: Ensure consistent environments across development and production.

Advanced Anomaly Detection:

Replace the simple threshold checks in QualityGate with an ML model (e.g., Isolation Forest) to detect subtle sensor drifts.
