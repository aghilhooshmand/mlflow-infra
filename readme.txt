ML-INFRA QUICK START
====================

1. Launch the Stack
-------------------

```
cd Deployment/ml-infra
docker compose --env-file .env build mlflow
docker compose --env-file .env up -d
```

Services will listen on:
- MLflow:       http://<host>:6001
- MinIO:        http://<host>:9000
- MinIO Console: http://<host>:9001
- Postgres URI:  postgres://user:pass@<host>:5432/mlflow

Use the admin credentials from `mlflow/mlflow-config.ini` (default `mlflowadmin / mlflowadminpass`) to sign in.

2. Use MLflow from Your Laptop
------------------------------

```
python3 -m venv .venv
source .venv/bin/activate
pip install --upgrade pip
pip install mlflow pandas numpy scikit-learn
# add torch, deap, etc. if your projects require them

export MLFLOW_TRACKING_URI=http://<host>:6001
export MLFLOW_TRACKING_USERNAME=<mlflow-username>
export MLFLOW_TRACKING_PASSWORD=<mlflow-password>
export MLFLOW_S3_ENDPOINT_URL=http://<host>:9000
```

You now have the client configured to log runs to the shared stack.

Quick sanity check (optional):
```
python - <<'PY'
import mlflow
mlflow.set_tracking_uri("http://<host>:6001")
mlflow.set_experiment("quick-test")
with mlflow.start_run(tags={"owner": "quickstart"}):
    mlflow.log_param("alpha", 0.1)
    mlflow.log_metric("rmse", 0.42)
print("Logged run to MLflow!")
PY
```
Replace `<host>` and credentials if you are running this from a different machine.

Need users or permissions?
```
# create user ( if you need add more user, already you have admin user that define in .env file)
docker compose --env-file .env exec mlflow \
  mlflow security create-user --username teammate --password StrongPass123! --admin false

# grant edit rights to an existing experiment
docker compose --env-file .env exec mlflow \
  mlflow security set-experiment-permission \
  --experiment-name "FORGE - Classic ML Experiments" \
  --username teammate \
  --permission edit
```

3. Need More Details?
---------------------

See `Deployment/DEPLOYMENT.md` for full explanations, optional tweaks (password changes, backups, etc.), and Git/DVC recommendations.

Otherwise, everything is ready—start writing your code and tracking experiments! 💡
