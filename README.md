# Real-Time Fraud Detection System

An end-to-end, production-style MLOps project that detects fraudulent transactions in real time using a streaming architecture. All transaction data is **synthetically generated** — no real data sources or credentials are used anywhere in this repository.

## Architecture

```
Transactions (synthetic producer)
        │
        ▼
     Kafka  ──────────────  topic: transactions
        │
        ▼
 Spark Structured Streaming  ──  real-time feature engineering
        │
        ▼
  Feature Store (in-memory / Redis-ready abstraction)
        │
        ▼
   Model Server (FastAPI + MLflow model)
        │
        ▼
  Fraud Prediction  ──  topic: fraud-predictions
        │
        ▼
   Dashboard (Grafana)
        │
        ▼
  Monitoring (Prometheus metrics, alerts, autoscaling signals)
```

## Tech Stack

| Layer | Technology |
|---|---|
| Ingestion | Apache Kafka |
| Stream processing | Apache Spark (Structured Streaming) |
| Model tracking | MLflow |
| Serving | FastAPI + Uvicorn |
| Containerization | Docker / Docker Compose |
| Orchestration | Kubernetes (Deployment, HPA, Canary) |
| Metrics | Prometheus |
| Dashboards | Grafana |

## Production Concepts Demonstrated

- **Streaming inference** — transactions scored as they arrive from Kafka
- **Real-time monitoring** — Prometheus metrics exposed by the model server, visualized in Grafana
- **Autoscaling** — Kubernetes HorizontalPodAutoscaler based on CPU / request load
- **Canary deployment** — traffic-split manifests for safely rolling out new model versions

## Project Structure

```
fraud-detection-system/
├── producer/            # Synthetic transaction generator → Kafka
├── streaming/           # Spark Structured Streaming job (feature engineering + scoring)
├── model/               # Model training + MLflow tracking
├── serving/             # FastAPI model server with Prometheus metrics
├── k8s/                 # Kubernetes manifests (deployment, HPA, canary)
├── monitoring/          # Prometheus + Grafana configs
├── docker/              # Dockerfiles
├── docker-compose.yml   # Local stack: Kafka, MLflow, model server, Prometheus, Grafana
└── requirements.txt
```

## Quickstart (Local, Docker Compose)

Prerequisites: Docker + Docker Compose.

```bash
# 1. Train a model (creates model/artifacts/fraud_model.pkl and logs to MLflow)
pip install -r requirements.txt
python model/train.py

# 2. Start the full stack
docker compose up --build -d

# 3. Start producing synthetic transactions
python producer/transaction_producer.py

# 4. (Optional) Run the Spark streaming job
python streaming/spark_streaming_job.py
```

Service endpoints:

| Service | URL |
|---|---|
| Model server (predict) | http://localhost:8000/predict |
| Model server health | http://localhost:8000/health |
| Prometheus metrics | http://localhost:8000/metrics |
| MLflow UI | http://localhost:5000 |
| Prometheus | http://localhost:9090 |
| Grafana | http://localhost:3000 (default login: admin / admin) |

### Example prediction request

```bash
curl -X POST http://localhost:8000/predict \
  -H "Content-Type: application/json" \
  -d '{
        "transaction_id": "txn-0001",
        "amount": 912.50,
        "hour_of_day": 3,
        "merchant_category": 5,
        "distance_from_home_km": 240.0,
        "txn_count_last_hour": 7,
        "avg_amount_last_24h": 85.0
      }'
```

Response:

```json
{"transaction_id": "txn-0001", "fraud_probability": 0.93, "is_fraud": true, "model_version": "v1"}
```

## Kubernetes Deployment

```bash
kubectl apply -f k8s/namespace.yaml
kubectl apply -f k8s/model-server-deployment.yaml
kubectl apply -f k8s/model-server-service.yaml
kubectl apply -f k8s/hpa.yaml

# Canary rollout of a new model version (10% traffic)
kubectl apply -f k8s/canary-deployment.yaml
```

The canary strategy runs `fraud-model-server-canary` alongside the stable deployment. Both share the same Service selector, so traffic splits proportionally to replica counts (9 stable : 1 canary ≈ 10%). Promote by scaling up the canary and scaling down stable; roll back by deleting the canary deployment.

## Monitoring

The model server exposes Prometheus metrics at `/metrics`:

- `fraud_predictions_total{result="fraud|legit"}` — prediction counts
- `prediction_latency_seconds` — inference latency histogram
- `fraud_probability` — distribution of predicted scores (drift signal)

Prometheus scrapes these and Grafana visualizes throughput, latency (p50/p95/p99), fraud rate, and score drift.

## Data Privacy

- No real transaction data, PII, or external data sources are used.
- All data is generated in-memory by `producer/transaction_producer.py`.
- No credentials, API keys, or secrets are committed. Configuration is via environment variables (see `.env.example`).

## License

MIT
