# 3. Pipeline Configuration

### 3.1 JSON Configuration Structure

```json
{
  "id": "my-pipeline-id",
  "name": "my-pipeline",
  "storage": "/mnt/my_storage/dlt",
  "configuration": {
    "spark.databricks.photon.enabled": "true",
    "spark.sql.shuffle.partitions": "200"
  },
  "clusters": [
    {
      "label": "default",
      "numWorkers": 2,
      "nodeTypeId": "i3.xlarge"
    }
  ],
  "libraries": [
    {
      "notebook": {
        "path": "/Repos/my-org/dlt-pipelines/notebook"
      }
    }
  ],
  "continuous": false,
  "development": false
}
```

**Key fields**:

- `storage`: Root path for output tables + checkpoints + event logs
- `configuration`: Spark configs (applied to all clusters)
- `continuous` vs `development` vs `production` modes (mutually exclusive scheduling)

### 3.2 Development vs Production Mode

| Aspect              | Development               | Production                  |
| ------------------- | ------------------------- | --------------------------- |
| Cluster persistence | Persists between updates  | Terminated after update     |
| Retry behavior      | Limited (faster feedback) | Full retry logic            |
| Cost                | Lower (shared clusters)   | Higher (dedicated clusters) |
| Observability       | Dashboard accessible      | Same as dev                 |
| Trigger             | Manual or scheduled       | Scheduled only              |

**Interview note**: Development mode speeds iteration; always validate in production mode before deploying.

### 3.3 Continuous vs Triggered Execution

- **Continuous**: Cluster always running, processes data as it arrives
- **Triggered**: Cluster starts only on schedule or manual trigger

Cost/latency tradeoff: Continuous = lower latency but constant DBU burn.

### 3.4 Autoscaling and Photon

```json
"autoscale": {
  "min_workers": 1,
  "max_workers": 10
}
```

**Photon**: Vectorized query execution engine

- 2-4x speedup for ETL workloads
- Higher cost (~2x DBU multiplier)
- Best for aggregations, joins; not I/O-bound operations

**When NOT to use Photon**: Data extraction, lightweight ingestion, single-node Python code.
