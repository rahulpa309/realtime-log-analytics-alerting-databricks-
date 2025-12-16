# 🚀 Real-Time Log Analytics & Alerting System

Production-style **real-time log monitoring pipeline** built using **PySpark Structured Streaming, Delta Lake, and Databricks Community Edition**. The project simulates Kafka-style ingestion, applies **data quality checks**, uses **SCD Type-2 dimensions**, and generates **real-time alerts and analytics**.

---

## 📌 Business Use Case

Modern applications generate massive volumes of logs. Operations teams need to:

* Detect error spikes in real time
* Monitor application performance
* Trigger alerts automatically
* Analyze historical trends for RCA (Root Cause Analysis)

This project simulates a **cloud-scale log analytics platform** used by SaaS companies, IT services, banks, and e-commerce platforms.

---

## 🏗️ Architecture Overview

```
Log Generator (Kafka-like)
        ↓
Databricks Auto Loader
        ↓
Bronze Delta Table (Raw Logs)
        ↓
Silver Delta Table (Clean + Validated Logs)
        ↓
SCD-2 Service Dimension
        ↓
Gold Delta Table (Aggregated Metrics)
        ↓
Alerts + Dashboards
```

---

## 🧰 Tech Stack

| Layer         | Technology                   |
| ------------- | ---------------------------- |
| Streaming     | PySpark Structured Streaming |
| Storage       | Delta Lake                   |
| Platform      | Databricks Community Edition |
| Data Quality  | Custom PySpark DQ Rules      |
| Modeling      | SCD Type-2                   |
| Visualization | Databricks SQL               |

---

## 📁 Project Structure

```
Real-Time-Log-Analytics/
│
├── notebooks/
│   ├── 00_log_generator
│   ├── 01_bronze_log_ingestion
│   ├── 02_silver_log_cleaning
│         ├── 02.1_data_quality_checks
│         ├── 02.2_scd2_service_metadata
│   ├── 03_gold_aggregations
│   ├── 04_alerting_logic
│   └── 05_dashboard_queries
│
└── README.md
```

---

## 🧾 Log Schema

```json
{
  "event_id": "uuid",
  "timestamp": "ISO-8601",
  "service": "payment-service",
  "level": "INFO | WARN | ERROR",
  "message": "string",
  "host": "server-01",
  "response_time_ms": 1200
}
```

---

## 🥉 Bronze Layer – Raw Ingestion

* Ingest logs continuously using Auto Loader
* Schema enforced, no transformation
* Fault-tolerant with checkpoints

**Output:** `bronze_logs`

---

## 🥈 Silver Layer – Cleaning & Standardization

* Convert timestamps
* Normalize log levels
* Add partition column (`log_date`)
* Drop malformed records

**Output:** `silver_logs`

---

## ✅ Data Quality Checks (Critical Feature)

### Why Data Quality?

In real systems, **bad data = bad alerts**. This layer ensures only valid data moves forward.

---

### 📏 Data Quality Rules Implemented

| Rule                  | Description                        |
| --------------------- | ---------------------------------- |
| Not Null              | `event_id`, `timestamp`, `service` |
| Valid Enum            | `level ∈ (INFO, WARN, ERROR)`      |
| Range Check           | `response_time_ms > 0`             |
| Freshness             | Timestamp not in future            |
| Referential Integrity | Service exists in dimension        |

---

### 🧪 DQ Implementation (PySpark)

```python
from pyspark.sql.functions import *

valid_levels = ['INFO', 'WARN', 'ERROR']

validated_df = (
    spark.readStream.table("silver_logs")
    .withColumn("dq_error",
        when(col("event_id").isNull(), "NULL_EVENT_ID")
        .when(col("service").isNull(), "NULL_SERVICE")
        .when(~col("level").isin(valid_levels), "INVALID_LEVEL")
        .when(col("response_time_ms") <= 0, "INVALID_RESPONSE_TIME")
        .when(col("timestamp") > current_timestamp(), "FUTURE_TIMESTAMP")
        .otherwise("VALID")
    )
)
```

---

### 🟢 Valid Records

```python
validated_df.filter(col("dq_error") == "VALID") \
  .writeStream \
  .format("delta") \
  .option("checkpointLocation", "/FileStore/checkpoints/silver_valid") \
  .table("silver_logs_valid")
```

---

### 🔴 Invalid Records (Quarantine)

```python
validated_df.filter(col("dq_error") != "VALID") \
  .writeStream \
  .format("delta") \
  .option("checkpointLocation", "/FileStore/checkpoints/quarantine") \
  .table("quarantine_logs")
```

**Benefit:** Bad data is isolated, not lost.

---

## 🧬 SCD Type-2 Service Dimension

Tracks historical changes in service ownership and criticality.

**Columns:**

* `service`
* `owner`
* `tier`
* `effective_from`
* `effective_to`
* `is_current`

Used to enrich logs with **current service metadata**.

---

## 🥇 Gold Layer – Aggregations

Metrics computed every **5-minute window**:

* Error count per service
* Average response time
* Log volume trends

**Output:** `gold_log_metrics`

---

## 🚨 Alerting Logic

### Alert Conditions

| Metric            | Threshold      |
| ----------------- | -------------- |
| ERROR count       | > 50 in 5 mins |
| Avg response time | > 2000 ms      |

Alerts are written to `alerts_table` and can be integrated with:

* Slack webhook
* Email

---

## 📊 Dashboards

Built using Databricks SQL:

* Errors by service
* Error trend over time
* Response time heatmap
* Top failing services

---

## ⚙️ Performance Optimizations

* Partitioning by `log_date`
* Watermarking for late data
* Delta Lake checkpointing
* Z-ORDER on `service`, `level`
* Broadcast join for service dimension

---

## 📈 Business Impact

* Faster incident detection
* Reduced MTTR (Mean Time to Resolution)
* Improved SLA compliance
* Production-ready streaming design

---

## ⭐ Future Enhancements

* Great Expectations integration
* Schema drift detection
* ML-based anomaly detection
* Cloud deployment (AWS/Azure)

---
