# 14. CI/CD & DevOps

### 14.1 Version Control & Code Organization

Typical structure:

```
repo/
├── pipelines/
│   ├── bronze.py       # Ingestion
│   ├── silver.py       # Transformation
│   └── gold.py         # Aggregation
├── tests/
│   ├── test_bronze.py
│   └── test_silver.py
├── config/
│   ├── dev.json        # Development pipeline config
│   ├── staging.json    # Staging config
│   └── prod.json       # Production config
└── .github/workflows/
    └── deploy.yml      # CI/CD workflow
```

### 14.2 Environment Promotion (Dev → Staging → Prod)

**Strategy**: Single codebase, different configs per environment.

```yaml
# .github/workflows/deploy.yml
name: Deploy DLT Pipeline
on:
  push:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - name: Run unit tests
        run: pytest tests/

  deploy-staging:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - name: Deploy to staging
        env:
          DATABRICKS_HOST: ${{ secrets.DATABRICKS_HOST }}
          DATABRICKS_TOKEN: ${{ secrets.DATABRICKS_TOKEN }}
        run: |
          databricks cli pipelines upsert \
            --config config/staging.json

  deploy-prod:
    needs: deploy-staging
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    steps:
      - uses: actions/checkout@v2
      - name: Deploy to production
        env:
          DATABRICKS_HOST: ${{ secrets.DATABRICKS_HOST }}
          DATABRICKS_TOKEN: ${{ secrets.DATABRICKS_TOKEN }}
        run: |
          databricks cli pipelines upsert \
            --config config/prod.json
```

### 14.3 Parameterization

```python
# pipeline.py
import dlt
import os

environment = os.getenv("ENVIRONMENT", "dev")
storage_path = f"s3://my-bucket/{environment}/data"

@dlt.table
def raw_data():
    return spark.readStream.format("cloudFiles") \
        .load(f"{storage_path}/raw/")
```

### 14.4 Testing Strategies

**Unit testing** (locally or in notebook):

```python
# tests/test_transformations.py
from pyspark.sql import SparkSession

spark = SparkSession.builder.master("local").appName("test").getOrCreate()

def test_data_quality():
    raw_df = spark.createDataFrame([
        (1, "Alice", 25),
        (2, "Bob", -5),  # Invalid age
    ], ["id", "name", "age"])

    # Simulate DLT expectation: age >= 0
    result = raw_df.filter("age >= 0")
    assert result.count() == 1  # Should drop one row
```

**Integration testing**: Deploy test pipeline to staging; validate outputs.
