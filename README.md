# Real-time Flight Delay Prediction on Kubernetes

End-to-end streaming ML system that predicts flight arrival delays in real time, deployed on **Google Kubernetes Engine**. Built by Miguel Vera and Pablo Melero for the Big Data course (IBDN) at ETSIT-UPM, starting from the course's base flight-prediction app.

A user submits a flight in the web app, the request travels through Kafka, a Spark Streaming job scores it with a model trained on an Iceberg lakehouse, and the prediction comes back to the browser over WebSockets in a few seconds.

## Architecture

```
 Flask web app ──► Kafka ──► Spark Streaming predictor (Scala) ──► Cassandra
      ▲                              │ loads model                     │
      └──────── WebSockets ◄─────────┼─────────────────────────────────┘
                                     │
 Airflow DAG ──► Spark training job (PySpark, Random Forest) ──► MLflow
                        │
                 Iceberg lakehouse on MinIO
```

| Component | Role |
|---|---|
| **Kafka** (KRaft) | Message queue between the web app and the predictor |
| **Spark** (master + workers, cluster mode) | Model training and streaming prediction |
| **Apache Iceberg on MinIO** | Lakehouse with the training data (S3-compatible storage) |
| **MLflow** | Experiment tracking and model versioning |
| **Airflow** | Orchestrates training with a `KubernetesPodOperator` DAG |
| **Cassandra** | Airport distances and stored predictions |
| **PostgreSQL** | Airflow metadata |
| **Flask + Socket.IO** | Prediction frontend |

## What we added on top of the base app

- Real-time results over **WebSockets** instead of polling.
- **Cassandra** for airport distances and prediction storage.
- **Iceberg lakehouse** on MinIO as the training data source, with **MLflow** tracking.
- **Spark in cluster mode** and an **Airflow DAG** for training.
- Full deployment on **GKE**: Kubernetes manifests for all services and three Bash scripts that provision the cluster, build and push the images and deploy everything in order.

## Run it

Requirements: a GCP project with billing enabled, `gcloud`, `kubectl` and Docker.

```bash
gcloud auth login
gcloud config set project <YOUR_PROJECT_ID>

bash scripts/01-setup-gke.sh       # GKE cluster (2 × e2-standard-4) + Artifact Registry
bash scripts/02-build-images.sh    # build and push the Spark and Flask images
bash resources/download_data.sh    # training data and airport distances
bash scripts/03-deploy.sh          # deploy, load data, train and start the predictor
```

The project ID (`practica-creativa-494612`) and zone (`europe-west1-b`) are hardcoded in the scripts and Kubernetes manifests. To use your own project:

```bash
grep -rl "practica-creativa-494612" --include="*.sh" --include="*.yaml" scripts/ k8s/ \
  | xargs sed -i "s/practica-creativa-494612/<YOUR_PROJECT_ID>/g"
```

Once deployed, get the external IPs with:

```bash
kubectl get services -n flight-prediction --field-selector spec.type=LoadBalancer
```

| Service | URL |
|---|---|
| Web app | `http://<FLASK_IP>/flights/delays/predict_kafka` |
| MLflow | `http://<MLFLOW_IP>:5050` |
| Airflow | `http://<AIRFLOW_IP>:8082` |
| Spark UI | `http://<SPARK_IP>:8080` |
| MinIO console | `http://<MINIO_IP>:9001` |

A `docker-compose.yml` is also included to run the whole stack locally.

## Clean up

```bash
gcloud container clusters delete flight-prediction-gke --zone europe-west1-b
gcloud artifacts repositories delete flight-prediction --location us-central1
```

## License

MIT
