---
name: data-engineering
description: Building data pipelines — ETL stages, Spark batch processing, quality gates, and star-schema warehouse modeling
---

# Data engineering

## An ETL pipeline

```python
from datetime import datetime
from dataclasses import dataclass

@dataclass
class PipelineResult:
    records_extracted: int
    records_transformed: int
    records_loaded: int
    errors: list[str]
    duration_seconds: float

class OrderPipeline:
    def __init__(self, source_db, warehouse_db):
        self.source = source_db
        self.warehouse = warehouse_db

    def extract(self, since: datetime) -> list[dict]:
        query = """
            SELECT o.*, c.name as customer_name, c.segment
            FROM orders o
            JOIN customers c ON o.customer_id = c.id
            WHERE o.updated_at > %s
        """
        return self.source.fetch_all(query, [since])

    def transform(self, records: list[dict]) -> list[dict]:
        transformed = []
        for record in records:
            transformed.append({
                "order_id": record["id"],
                "customer_name": record["customer_name"],
                "segment": record["segment"].upper(),
                "total_amount": float(record["total"]),
                "order_date": record["created_at"].date(),
                "fiscal_quarter": get_fiscal_quarter(record["created_at"]),
                "is_high_value": float(record["total"]) > 1000,
                "loaded_at": datetime.utcnow(),
            })
        return transformed

    def load(self, records: list[dict]) -> int:
        return self.warehouse.upsert_batch(
            table="fact_orders",
            records=records,
            conflict_keys=["order_id"],
            batch_size=5000,
        )

    def run(self, since: datetime) -> PipelineResult:
        start = datetime.utcnow()
        raw = self.extract(since)
        clean = self.transform(raw)
        loaded = self.load(clean)
        return PipelineResult(
            records_extracted=len(raw),
            records_transformed=len(clean),
            records_loaded=loaded,
            errors=[],
            duration_seconds=(datetime.utcnow() - start).total_seconds(),
        )
```

## Batch processing with Spark

```python
from pyspark.sql import SparkSession
from pyspark.sql import functions as F
from pyspark.sql.window import Window

spark = SparkSession.builder \
    .appName("sales-analytics") \
    .config("spark.sql.adaptive.enabled", "true") \
    .config("spark.sql.shuffle.partitions", "200") \
    .getOrCreate()

orders = spark.read.parquet("s3://data-lake/orders/")
customers = spark.read.parquet("s3://data-lake/customers/")

daily_revenue = (
    orders
    .filter(F.col("status") == "completed")
    .withColumn("order_date", F.to_date("created_at"))
    .groupBy("order_date", "product_category")
    .agg(
        F.sum("total_amount").alias("revenue"),
        F.count("id").alias("order_count"),
        F.avg("total_amount").alias("avg_order_value"),
    )
    .withColumn(
        "revenue_7d_avg",
        F.avg("revenue").over(
            Window.partitionBy("product_category")
            .orderBy("order_date")
            .rowsBetween(-6, 0)
        )
    )
)

daily_revenue.write \
    .partitionBy("order_date") \
    .mode("overwrite") \
    .parquet("s3://data-warehouse/daily_revenue/")
```

## Quality gates

```python
from dataclasses import dataclass

@dataclass
class QualityCheck:
    name: str
    query: str
    threshold: float
    severity: str

CHECKS = [
    QualityCheck(
        name="null_customer_ids",
        query="SELECT COUNT(*) FROM fact_orders WHERE customer_id IS NULL",
        threshold=0,
        severity="critical",
    ),
    QualityCheck(
        name="negative_amounts",
        query="SELECT COUNT(*) FROM fact_orders WHERE total_amount < 0",
        threshold=0,
        severity="critical",
    ),
    QualityCheck(
        name="duplicate_orders",
        query="SELECT COUNT(*) - COUNT(DISTINCT order_id) FROM fact_orders",
        threshold=0,
        severity="warning",
    ),
    QualityCheck(
        name="freshness",
        query="SELECT EXTRACT(EPOCH FROM NOW() - MAX(loaded_at))/3600 FROM fact_orders",
        threshold=2.0,
        severity="warning",
    ),
]

def run_quality_checks(db, checks: list[QualityCheck]) -> list[dict]:
    results = []
    for check in checks:
        value = db.fetch_scalar(check.query)
        passed = value <= check.threshold
        results.append({
            "name": check.name,
            "value": value,
            "threshold": check.threshold,
            "passed": passed,
            "severity": check.severity,
        })
        if not passed and check.severity == "critical":
            raise DataQualityError(f"Critical check failed: {check.name} = {value}")
    return results
```

## A star-schema warehouse

```sql
CREATE TABLE dim_customers (
    customer_key    BIGINT PRIMARY KEY,
    customer_id     VARCHAR(50) NOT NULL,
    name            VARCHAR(200),
    segment         VARCHAR(50),
    country         VARCHAR(100),
    valid_from      TIMESTAMP NOT NULL,
    valid_to        TIMESTAMP,
    is_current      BOOLEAN DEFAULT TRUE
);

CREATE TABLE dim_products (
    product_key     BIGINT PRIMARY KEY,
    product_id      VARCHAR(50) NOT NULL,
    name            VARCHAR(200),
    category        VARCHAR(100),
    subcategory     VARCHAR(100)
);

CREATE TABLE fact_orders (
    order_key       BIGINT PRIMARY KEY,
    order_id        VARCHAR(50) UNIQUE NOT NULL,
    customer_key    BIGINT REFERENCES dim_customers(customer_key),
    product_key     BIGINT REFERENCES dim_products(product_key),
    order_date_key  INT,
    quantity        INT,
    unit_price      DECIMAL(10,2),
    total_amount    DECIMAL(12,2),
    loaded_at       TIMESTAMP DEFAULT NOW()
);
```

## What to avoid

- Crunching data row by row instead of in batches or sets
- Leaving large tables unpartitioned by date or category
- No quality checks between pipeline stages
- Dumping raw data straight into the warehouse with no transformation
- Full table scans where an incremental load would do
- No lineage tracking — where data came from and when it ran

## Before you ship

- [ ] Pipelines separate extract, transform, and load into clear stages
- [ ] Processing is incremental via watermarks or change data capture
- [ ] Quality checks run after each stage
- [ ] The warehouse uses a star or snowflake schema of dimensions and facts
- [ ] Spark jobs enable adaptive query execution and partition sensibly
- [ ] Loads are idempotent — a rerun yields the same result
- [ ] Data freshness is monitored with automated alerts
- [ ] Schema changes stay additive and are handled gracefully
