---
title: Streaming PostgreSQL Updates to Kafka with Debezium
tags:
- k8s
date: 2020-10-02 15:39:00
---

## setup
```shell
kubectl create namespace kafka-connect-tutorial
kubectl config set-context --current --namespace kafka-connect-tutorial # optional
```

## kafka

```
helm install kafka --namespace kafka-connect-tutorial apphub/kafka --set external.enabled=true,global.storageClass=fast,persistence.size=1Gi
```

## Kafka Connect client

```yaml
cat > kafka-client-deploy.yaml <<EOF
# kafka-client-deploy.yaml
apiVersion: v1
kind: Pod
metadata:
  name: kafka-client
spec:
  containers:
  - name: kafka-client
    image: confluentinc/cp-kafka:5.0.1
    command:
      - sh
      - -c
      - "exec tail -f /dev/null"
EOF
```

```shell
kubectl create -f kafka-client-deploy.yaml -n kafka-connect-tutorial
kubectl -n kafka-connect-tutorial exec kafka-client -- kafka-topics --zookeeper kafka-zookeeper:2181 --topic connect-offsets --create --partitions 1 --replication-factor 1
kubectl -n kafka-connect-tutorial exec kafka-client -- kafka-topics --zookeeper kafka-zookeeper:2181 --topic connect-configs --create --partitions 1 --replication-factor 1
kubectl -n kafka-connect-tutorial exec kafka-client -- kafka-topics --zookeeper kafka-zookeeper:2181 --topic connect-status --create --partitions 1 --replication-factor 1
```

## Kafka Connect

```yaml
# kafka-connect-deploy.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: kafkaconnect-deploy
  labels:
    app: kafkaconnect
spec:
  replicas: 1
  selector:
    matchLabels:
      app: kafkaconnect
  template:
    metadata:
      labels:
        app: kafkaconnect
    spec:
      containers:
        - name: kafkaconnect-container
          image: debezium/connect:latest
          readinessProbe:
            httpGet:
              path: /
              port: 8083
          livenessProbe:
            httpGet:
              path: /
              port: 8083
          env:
            - name: BOOTSTRAP_SERVERS
              value: kafka-headless.kafka:9092
            - name: GROUP_ID
              value: "1"
            - name: OFFSET_STORAGE_TOPIC
              value: connect-offsets
            - name: CONFIG_STORAGE_TOPIC
              value: connect-configs
            - name: STATUS_STORAGE_TOPIC
              value: connect-status
          ports:
          - containerPort: 8083
---
apiVersion: v1
kind: Service
metadata:
  name: kafkaconnect-service
  labels:
    app: kafkaconnect-service
spec:
  type: NodePort
  ports:
    - name: kafkaconnect
      protocol: TCP
      port: 8083
      nodePort: 30500
  selector:
      app: kafkaconnect

```

```shell
kubectl apply -f kafka-connect-deploy.yaml -n kafka-connect-tutorial
```

## pg

```
# extended.conf

wal_level = logical
max_wal_senders = 1
max_replication_slots = 1
```

```shell
kubectl create configmap --namespace kafka-connect-tutorial --from-file=extended.conf postgresql-config
helm install postgres --namespace kafka-connect-tutorial --set extendedConfConfigMap=postgresql-config --set service.type=NodePort --set service.nodePort=30600 --set postgresqlPassword=passw0rd,global.storageClass=fast,persistence.size=1Gi apphub/postgresql
```

```shell
kubectl exec --namespace kafka-connect-tutorial -it postgres-postgresql-0  -- /bin/sh
psql --user postgres
# => 导入测试数据

# 绑定
curl -X POST \
  http://192.168.1.61:30500/connectors \
  -H 'Content-Type: application/json' \
  -d '{
    "name": "containers-connector",
    "config": {
            "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
            "plugin.name": "pgoutput",
            "database.hostname": "postgres-postgresql-headless.postgres",
            "database.port": "5432",
            "database.user": "postgres",
            "database.password": "postgres",
            "database.dbname": "mmtip-production",
            "database.server.name": "postgres"
      }
}'

kubectl -n kafka-connect-tutorial exec kafka-client -- kafka-topics --zookeeper kafka-zookeeper:2181 --list
kubectl -n kafka-connect-tutorial exec kafka-client -- kafka-console-consumer --topic postgres.public.containers --from-beginning --bootstrap-server kafka:9092
```

