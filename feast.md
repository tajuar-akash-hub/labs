# Dockerizing Feast — Hands-On Lab
---

## Goal

By the end you will:
- Know what Docker does and why we use it with Feast
- Write a Dockerfile that installs Feast
- Run Feast inside a container using Docker Compose
- Save your feature data to a folder on your computer so it does not get lost
- Use SQLite as the online store (no Redis needed)

---

## What is Feast?

Feast is a tool that stores ML features in two places:
- **Offline store** : big historical data (we use a Parquet file)
- **Online store** : small and fast (we use SQLite)

You read offline data when training a model, and online data when the model is making predictions.

## What is Docker?

Docker lets you package an app together with everything it needs (Python, libraries, files) into one box called a **container**. The container runs the same on any computer.

## Why put Feast in Docker?

- You do not need to install Python or Feast on your machine
- The setup is the same for you and your teammates
- When you break something, just delete the container and start again
- Easy to move the whole thing to a server later

---

## What we are building

```
Your computer
 └── Docker
      └── feast container
           ├── Python
           ├── Feast
           ├── features.py
           ├── feature_store.yaml
           └── data folder (shared with your computer)
```

Only one container. SQLite lives inside it as a file.

---

## ✅ Prerequisites

Check your environment:

```bash
docker --version
docker compose version
```

You should see something like:
```
Docker version 24.x.x
Docker Compose version v2.x.x
```


---

## 📂 Project Structure

Create this folder layout:

```
feast-docker-lab/
├── compose.yaml
├── Dockerfile
├── feature_store.yaml
├── features.py
├── test_workflow.py
├── data/
│   └── driver_stats.parquet
└── .env
```

---

## Step 1 — Create the Project

```bash
mkdir feast-docker-lab && cd feast-docker-lab
mkdir -p data
```

We'll build everything by hand so you understand each piece.

---

## Step 2 — Create the Feature Definitions

Create `features.py`:

```python
from datetime import timedelta
from feast import Entity, FeatureView, Field, FileSource
from feast.types import Float64, Int64, String

# ---------- Entities ----------
driver = Entity(name="driver_id", join_keys=["driver_id"])

# ---------- Data Source ----------
driver_stats_source = FileSource(
    name="driver_stats_source",
    path="data/driver_stats.parquet",
    timestamp_field="event_timestamp",
)

# ---------- Feature View ----------
driver_stats_fv = FeatureView(
    name="driver_hourly_stats",
    entities=[driver],
    ttl=timedelta(hours=2),
    schema=[
        Field(name="conv_rate", dtype=Float64),
        Field(name="acc_rate", dtype=Float64),
        Field(name="avg_daily_trips", dtype=Int64),
        Field(name="country", dtype=String),
    ],
    source=driver_stats_source,
    online=True,
)
```

This file defines **one entity** (`driver_id`) and **one feature view** (`driver_hourly_stats`) with 4 features.

---

## Step 3 — Create the Feature Store Config

Create `feature_store.yaml`:

```yaml
project: feast_docker_demo
registry: data/registry.db
provider: local
entity_key_serialization_version: 3

online_store:
  type: redis
  connection_string: "redis:6379"

offline_store:
  type: file
```

**Note:** `redis:6379` (not `localhost:6379`) — this is the **Docker service name** we will define in `compose.yaml`. Containers resolve it via Docker's internal DNS.

---

## Step 4 — Generate Sample Data

Create `data/driver_stats.parquet` using a small Python script (run locally or inside a temp container):

```python
# generate_data.py
import pandas as pd
from datetime import datetime, timedelta

rows = []
drivers = [1001, 1002, 1003, 1004, 1005]
countries = ["US", "BD", "IN", "UK", "CA"]

now = datetime.utcnow()
for i in range(100):
    rows.append({
        "driver_id": drivers[i % len(drivers)],
        "event_timestamp": now - timedelta(hours=i),
        "created": now - timedelta(hours=i),
        "conv_rate": round(0.7 + (i % 10) * 0.01, 3),
        "acc_rate": round(0.5 + (i % 7) * 0.02, 3),
        "avg_daily_trips": 50 + (i % 25),
        "country": countries[i % len(countries)],
    })

df = pd.DataFrame(rows)
df.to_parquet("data/driver_stats.parquet", index=False)
print("Created driver_stats.parquet with", len(df), "rows")
```

Run it:
```bash
pip install pandas pyarrow
python generate_data.py
```

---

## Step 5 — Create the Test Workflow

Create `test_workflow.py`:

```python
import subprocess
import sys

def run(cmd):
    print(f"\n>>> {cmd}")
    result = subprocess.run(cmd, shell=True, check=True)

def main():
    # 1. Apply feature definitions
    run("feast apply")

    # 2. Materialize features to online store
    run("feast materialize-incremental $(date -u +'%Y-%m-%dT%H:%M:%S')")

    # 3. Read online features
    run("feast online-features retrieve "
        "\"driver_id=1001 driver_id=1002\" "
        "\"driver_hourly_stats:conv_rate driver_hourly_stats:country\"")

if __name__ == "__main__":
    main()
```

This script will run **inside the Feast container**.

---

## Step 6 — Create the Dockerfile

Create `Dockerfile`:

```dockerfile
# Use official Python slim base
FROM python:3.10-slim

# Avoid Python writing .pyc files and buffering stdout
ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1

# Set workdir
WORKDIR /app

# Install system dependencies (build tools for some pip packages)
RUN apt-get update && apt-get install -y --no-install-recommends \
        build-essential \
        curl \
    && rm -rf /var/lib/apt/lists/*

# Install Feast + dependencies
RUN pip install --no-cache-dir \
        "feast[redis]" \
        pandas \
        pyarrow \
        redis

# Copy project files
COPY feature_store.yaml .
COPY features.py .
COPY test_workflow.py .
COPY data ./data

# Default command — keep container alive
CMD ["tail", "-f", "/dev/null"]
```

**Why `tail -f /dev/null`?** It keeps the container running forever so you can exec commands inside it. We'll launch the Feast server separately when needed.

---

## Step 7 — Create the Docker Compose File

Create `compose.yaml`:

```yaml
services:
  redis:
    image: redis:7-alpine
    container_name: feast-redis
    ports:
      - "6379:6379"
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 5

  feast:
    build: .
    container_name: feast-server
    depends_on:
      redis:
        condition: service_healthy
    environment:
      - FEAST_USAGE=False      # disable telemetry
      - PYTHONPATH=/app
    volumes:
      - ./data:/app/data        # persist registry + parquet
      - ./feature_store.yaml:/app/feature_store.yaml
      - ./features.py:/app/features.py
    working_dir: /app
    ports:
      - "6566:6566"             # Feast feature server
      - "8888:8888"             # Feast UI (optional)
    command: tail -f /dev/null
```

---

## Step 8 — Build and Run

```bash
docker compose build
docker compose up -d
```

Check both containers are running:
```bash
docker compose ps
```

Expected:
```
NAME           STATUS          PORTS
feast-redis    Up (healthy)    0.0.0.0:6379->6379/tcp
feast-server   Up              0.0.0.0:6566->6566/tcp, 0.0.0.0:8888->8888/tcp
```

---

## Step 9 — Apply Features Inside the Container

```bash
docker compose exec feast feast apply
```

Expected output:
```
Registered entity driver_id
Registered feature view driver_hourly_stats
Deploying infrastructure for driver_hourly_stats
```

---

## Step 10 — Run the Test Workflow

```bash
docker compose exec feast python test_workflow.py
```

You should see:
- `No changes to registry` (because apply already ran)
- Materialization logs
- Online feature retrieval output:
```
driver_id  driver_hourly_stats__conv_rate  driver_hourly_stats__country
1001       0.71                            US
1002       0.72                            BD
```

---

## Step 11 — Start the Feast Feature Server

In one terminal:
```bash
docker compose exec feast feast serve --host 0.0.0.0 --port 6566
```

In another terminal, query it via HTTP:
```bash
curl -X POST http://localhost:6566/get-online-features \
  -H "Content-Type: application/json" \
  -d '{
    "features": [
      "driver_hourly_stats:conv_rate",
      "driver_hourly_stats:country"
    ],
    "entities": {
      "driver_id": ["1001", "1002"]
    }
  }'
```

Expected response (JSON):
```json
{
  "metadata": { "feature_names": ["driver_id", "driver_hourly_stats__conv_rate", ...] },
  "results": [
    { "values": ["1001", "1002"], "statuses": ["PRESENT", "PRESENT"] },
    { "values": [0.71, 0.72], "statuses": ["PRESENT", "PRESENT"] }
  ]
}
```

---

## Step 12 — Start the Feast UI (Optional)

```bash
docker compose exec feast feast ui --host 0.0.0.0 --port 8888
```

Open `http://localhost:8888` in your browser to browse features visually.

---

## Step 13 — Persist Data Across Restarts

The `volumes:` section in `compose.yaml` ensures the `data/` folder is mounted. To verify:

```bash
ls -la data/
```

You should see:
```
registry.db
driver_stats.parquet
```

Stop and restart:
```bash
docker compose down
docker compose up -d
docker compose exec feast feast entities list
```

Your features should still be registered — the registry survived the restart.

---

## Step 14 — Clean Up

Stop containers but keep data:
```bash
docker compose down
```

Stop + remove everything (including volumes):
```bash
docker compose down -v
docker image prune -f
```

---



## 🐞 Common Errors & Fixes

| Error | Cause | Fix |
|---|---|---|
| `Cannot connect to redis:6379` | Redis not ready yet | Add `depends_on` with `condition: service_healthy` |
| `No changes to registry` | Already applied | Harmless — proceed |
| `ModuleNotFoundError: feast` | Forgot to `pip install feast[redis]` in Dockerfile | Rebuild image |
| `Port 6566 already in use` | Another app using the port | Change port mapping: `"6567:6566"` |
| `Permission denied on data/` | Files owned by root inside container | Add `user: "1000:1000"` to compose service |

---



---

## ✅ Conclusion

You've just:

- ✅ Containerized a **complete Feast stack** with Redis
- ✅ Built a **custom Docker image** for Feast
- ✅ Used **Docker Compose** to orchestrate services
- ✅ Ran **feast apply** + materialization + online retrieval inside a container
- ✅ Started a **Feast feature server** reachable via HTTP
- ✅ Learned how volumes **persist state** across restarts

You now have a **reproducible, portable Feast setup** that you can run on any machine with Docker - no more "works on my laptop" issues.

---

