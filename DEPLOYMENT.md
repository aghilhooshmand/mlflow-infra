# ML-Infra Deployment (Local & Intranet)

This guide explains how to deploy the MLflow + Postgres + MinIO stack contained in `Deployment/ml-infra`. Follow the same steps whether you are setting it up on your workstation or on an internal Linux server.

---

## 1. Requirements

Install once on the host that will run the containers:

- Docker Engine (24+)
- Docker Compose v2
- Git

Ubuntu/Debian quick setup:

```bash
sudo apt-get update
sudo apt-get install -y docker.io docker-compose-plugin git
sudo systemctl enable --now docker
sudo usermod -aG docker $USER   # log out/in afterwards
```

Optional (only if you plan to run notebooks or CLI locally):
```bash
sudo apt-get install -y python3 python3-venv python3-pip
```

---

## 2. Directory & Files

```
Deployment/ml-infra/
├── docker-compose.yaml
├── mlflow/
│   ├── Dockerfile
│   └── config/mlflow-config.ini
├── postgres/
│   ├── init/init.sql
│   └── data/                  # created automatically
└── minio/
    ├── data/                  # created automatically
    └── certs/                 # leave empty unless you add TLS certs
```

`mlflow-config.ini` defines the default admin user/password and the Postgres URI used for MLflow’s security database.

---

## 3. Prepare the `.env`

In `Deployment/ml-infra`, create `.env`. Example values:

```env
# MinIO
MINIO_ROOT_USER=minioadmin
MINIO_ROOT_PASSWORD=StrongMinioPass!
MINIO_HOST_PORT=9000
MINIO_CONT_PORT=9000
MINIO_HOST_CONSOLE_PORT=9001
MINIO_CONT_CONSOLE_PORT=9001
MINIO_HOST_DATA_DIR=./minio/data
MINIO_CONT_DATA_DIR=/data
MINIO_HOST_CERT_DIR=./minio/certs
MINIO_CONT_CERT_DIR=/certs

# Postgres
POSTGRES_USER=postgres
POSTGRES_PASSWORD=postgres123!
POSTGRES_DB=mlflow
POSTGRES_HOST_PORT=5432
POSTGRES_CONT_PORT=5432

# MLflow server
MLFLOW_HOST_PORT=6001
MLFLOW_CONT_PORT=5000
MLFLOW_HOST_CONFIG_PATH=./mlflow/config/mlflow-config.ini
MLFLOW_CONT_CONFIG_PATH=/mlflow/config/mlflow-config.ini
```

> If you do **not** plan to provide TLS certificates, either leave `minio/certs` empty or edit `docker-compose.yaml` to remove the cert volume and change the `minio` command to `server ${MINIO_CONT_DATA_DIR}`.

Remember that the included `init.sql` creates two databases: `mlflow` (tracking data) and `mlflow_users` (MLflow auth metadata). No extra action is required beyond supplying the credentials above.

Feel free to change passwords or ports—just keep the host and container paths aligned with the compose file.

---

## 4. Launch the Stack

```bash
cd Deployment/ml-infra
docker compose --env-file .env build mlflow
docker compose --env-file .env up -d
```

After the containers start:

1. Log into the MinIO console (`http://<host>:9001`) with `MINIO_ROOT_USER / MINIO_ROOT_PASSWORD` and create a bucket named `mlflow` if it does not already exist. This is where MLflow will store artifacts.
2. Visit `http://<host>:6001` and sign in with the admin credentials from `mlflow-config.ini` (default `mlflowadmin / mlflowadminpass`).

Services are now available at:

| Service  | URL                         |
|----------|-----------------------------|
| MLflow   | `http://<host>:6001`        |
| MinIO    | `http://<host>:9000`        |
| MinIO UI | `http://<host>:9001`        |
| Postgres | `postgres://user:pass@<host>:5432/mlflow` |

Use the admin credentials from `mlflow-config.ini` (`mlflowadmin` / `mlflowadminpass` by default) to sign into MLflow.

---

## 5. Managing Users & Passwords

- To change the MLflow admin username/password, edit `mlflow/config/mlflow-config.ini` and restart the container:
  ```bash
  docker compose --env-file .env restart mlflow
  ```
- To add more MLflow users, use the CLI once the server is running:
  ```bash
  docker compose exec mlflow mlflow security create-user \
    --username analyst --password StrongPass! --admin false
  ```
  Other helpful commands:
  ```bash
  docker compose exec mlflow mlflow security list-users
  docker compose exec mlflow mlflow security update-user --username analyst --password NewPass!
  ```
- MinIO credentials come from `.env` (`MINIO_ROOT_USER` / `MINIO_ROOT_PASSWORD`). Change them there and restart `minio` if needed.
- Postgres credentials are controlled by `POSTGRES_USER` / `POSTGRES_PASSWORD` in `.env`.

---

## 6. Using MLflow from Your Laptop

You do **not** need to run another MLflow server locally. Install only the client libraries you use and point them to the running instance:

```bash
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

Any script or notebook that calls `mlflow.log_*` will now save runs to the shared server.

Consider using Git and DVC to version your code and datasets alongside your MLflow runs:

```bash
git init
dvc init
dvc add data/raw_dataset.csv
git add data/raw_dataset.csv.dvc .gitignore notebooks/
git commit -m "Initial experiment"
```

---

## 7. Routine Commands

| Action | Command |
|--------|---------|
| Check status | `docker compose --env-file .env ps` |
| View logs    | `docker compose --env-file .env logs -f mlflow` |
| Restart MLflow | `docker compose --env-file .env restart mlflow` |
| Stop everything | `docker compose --env-file .env down` |
| Stop + remove data (irreversible) | `docker compose --env-file .env down -v` |

**Postgres backup / restore:**
```bash
docker compose exec postgres pg_dump -U "$POSTGRES_USER" "$POSTGRES_DB" > backup.sql
docker compose exec -T postgres psql -U "$POSTGRES_USER" "$POSTGRES_DB" < backup.sql
```

(Optional) to snapshot the MLflow auth DB instead:

```bash
docker compose exec postgres pg_dump -U "$POSTGRES_USER" mlflow_users > backup_users.sql
```

### Managing Users and Permissions via CLI

MLflow reads additional accounts from the auth database, so use the built-in CLI after deployment:

```bash
# create a non-admin user
cd Deployment/ml-infra
docker compose --env-file .env exec mlflow \
  mlflow security create-user --username forge_user --password StrongPass123! --admin false

# list all users
docker compose --env-file .env exec mlflow mlflow security list-users

# grant edit permission on an experiment
docker compose --env-file .env exec mlflow \
  mlflow security set-experiment-permission \
  --experiment-name "FORGE - Classic ML Experiments" \
  --username forge_user \
  --permission edit
```

Update or remove a user with:

```bash
# change password
docker compose --env-file .env exec mlflow \
  mlflow security update-user --username forge_user --password NewPass!

# delete user
docker compose --env-file .env exec mlflow \
  mlflow security delete-user --username forge_user
```

This approach keeps the admin credentials in `mlflow-config.ini` but lets you manage the rest of the team from the shell.

MinIO data lives on the host under `minio/data`; copy or snapshot this folder if you need off-host backups.

---

## 8. Deploying on an Intranet Server

Repeat the same steps on your internal server:

```bash
ssh user@server
cd /opt/forge/Deployment/ml-infra
# copy .env or create a new one with server-specific ports
docker compose --env-file .env build mlflow
docker compose --env-file .env up -d
```

Colleagues can now reach MLflow at `http://SERVER_IP:6001` (inside your intranet). No TLS, DNS, or cloud services are required unless you decide to expose it publicly later.

That’s all—MLflow, Postgres, and MinIO are ready for your team’s experiments with a single Compose stack. Keep `.env` and `mlflow-config.ini` safe, update credentials when staff changes, and use Git/DVC to keep projects reproducible.
