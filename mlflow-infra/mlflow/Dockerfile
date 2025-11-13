FROM python:3.13.1-slim

RUN apt-get update && apt-get upgrade -y && apt-get install -y curl \
    && rm -rf /var/lib/apt/lists/*
RUN pip install --no-cache-dir mlflow[extras] psycopg2-binary boto3 cryptography pymysql

EXPOSE 5000