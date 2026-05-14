# Deployment Architecture

## Overview

This document describes the production deployment architecture for the CEX matching engine, including Kubernetes topology, resource allocation, and operational procedures.

## Kubernetes Topology

```mermaid
flowchart TB
    subgraph K8s["Kubernetes Cluster"]
        subgraph Ingress["Ingress Layer"]
            NGINX["NGINX Ingress<br/>Controller"]
        end

        subgraph Gateway["API Gateway Namespace"]
            GW1["gateway-1"]
            GW2["gateway-2"]
            GW3["gateway-3"]
        end

        subgraph Engine["Matching Engine Namespace"]
            ME1["engine-shard-1-3<br/>CPU: 4c, RAM: 8Gi"]
            ME2["engine-shard-4-6<br/>CPU: 4c, RAM: 8Gi"]
            ME3["engine-shard-7-9<br/>CPU: 4c, RAM: 8Gi"]
        end

        subgraph Services["Backend Services"]
            Balance["balance-service<br/>3 replicas"
            Settle["settlement-service<br/>3 replicas"]
            MarketData["marketdata-service<br/>3 replicas"]
            History["history-service<br/>2 replicas"]
        end

        subgraph Infra["Infrastructure"]
            NATS1["nats-0"]
            NATS2["nats-1"]
            NATS3["nats-2"]

            Redis1["redis-0"]
            Redis2["redis-1"]
            Redis3["redis-2"]

            PG1["postgres-0"]
            PG2["postgres-1"]
        end

        subgraph Observability["Observability"]
            Prom["prometheus"
            Graf["grafana"
            Tempo["tempo"
            Loki["loki"
        end
    end

    NGINX --> GW1
    NGINX --> GW2
    NGINX --> GW3

    GW1 --> ME1
    GW2 --> ME2
    GW3 --> ME3

    ME1 --> NATS1
    ME2 --> NATS2
    ME3 --> NATS3

    NATS1 --> Balance
    NATS2 --> Balance
    NATS3 --> Balance

    Balance --> PG1
    Balance --> PG2

    GW1 --> Redis1
    GW2 --> Redis2
    GW3 --> Redis3

    classDef engine fill:#2563eb,stroke:#1e40af,color:#fff
    classDef infra fill:#059669,stroke:#047857,color:#fff
    classDef obs fill:#7c3aed,stroke:#6d28d9,color:#fff

    class ME1,ME2,ME3 engine
    class NATS1,NATS2,NATS3,Redis1,Redis2,Redis3,PG1,PG2 infra
    class Prom,Graf,Tempo,Loki obs
```

## Namespace Organization

```
exchange-production/
├── exchange-gateway/          # API Gateway pods
├── exchange-engine/           # Matching Engine pods
├── exchange-backend/          # Backend services
├── exchange-infra/            # Infrastructure (NATS, Redis)
├── exchange-database/         # PostgreSQL (via operator)
├── exchange-observability/    # Monitoring stack
└── exchange-cicd/             # CI/CD resources
```

## Resource Allocation

### Matching Engine Pods

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: engine-shard-1
  namespace: exchange-engine
spec:
  containers:
  - name: engine
    image: exchange/engine:v1.0.0
    resources:
      requests:
        cpu: "4"
        memory: "8Gi"
      limits:
        cpu: "4"
        memory: "8Gi"
    env:
    - name: SHARD_ID
      value: "1"
    - name: SHARD_SYMBOLS
      value: "BTC/USDT,ETH/USDT"
    - name: RUST_LOG
      value: "info,exchange_engine=debug"
    volumeMounts:
    - name: snapshot
      mountPath: /data/snapshots
    - name: config
      mountPath: /etc/exchange
      readOnly: true
  volumes:
  - name: snapshot
    persistentVolumeClaim:
      claimName: engine-snapshots-pvc
  - name: config
    configMap:
      name: engine-config
  nodeSelector:
    node.kubernetes.io/instance-type: "c5.xlarge"  # AWS
  affinity:
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: app
            operator: In
            values: [engine]
        topologyKey: "kubernetes.io/hostname"
```

### CPU Pinning

```yaml
spec:
  containers:
  - name: engine
    resources:
      limits:
        cpu: "4"
        memory: "8Gi"
    # Enable CPU pinning via guaranteed QoS
```

The matching engine uses **guaranteed QoS** by setting `requests == limits`, which enables:
- Exclusive CPU allocation
- No CPU throttling
- Minimal latency variance

### API Gateway Pods

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: gateway
  namespace: exchange-gateway
spec:
  replicas: 3
  selector:
    matchLabels:
      app: gateway
  template:
    metadata:
      labels:
        app: gateway
    spec:
      containers:
      - name: gateway
        image: exchange/gateway:v1.0.0
        resources:
          requests:
            cpu: "2"
            memory: "4Gi"
          limits:
            cpu: "2"
            memory: "4Gi"
        ports:
        - containerPort: 8080
          name: http
        - containerPort: 8081
          name: metrics
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: database-credentials
              key: url
        - name: REDIS_URL
          valueFrom:
            secretKeyRef:
              name: redis-credentials
              key: url
        - name: NATS_URL
          value: "nats://nats.exchange-infra.svc.cluster.local:4222"
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
```

## Infrastructure Components

### NATS JetStream Cluster

```yaml
apiVersion: v1
kind: StatefulSet
metadata:
  name: nats
  namespace: exchange-infra
spec:
  serviceName: nats
  replicas: 3
  selector:
    matchLabels:
      app: nats
  template:
    metadata:
      labels:
        app: nats
    spec:
      containers:
      - name: nats
        image: nats:alpine
        ports:
        - containerPort: 4222
          name: client
        - containerPort: 6222
          name: cluster
        - containerPort: 8222
          name: monitor
        args:
        - "--jetstream"
        - "-m"
        - "8222"
        - "--cluster_name"
        - "exchange-nats"
        - "--cluster"
        - "nats://0.0.0.0:6222"
        - "--routes"
        - "nats://nats-0.nats.exchange-infra.svc.cluster.local:6222"
        - "--routes"
        - "nats://nats-1.nats.exchange-infra.svc.cluster.local:6222"
        - "--routes"
        - "nats://nats-2.nats.exchange-infra.svc.cluster.local:6222"
        resources:
          requests:
            cpu: "2"
            memory: "4Gi"
          limits:
            cpu: "4"
            memory: "8Gi"
        volumeMounts:
        - name: jetstream
          mountPath: /data/jetstream
  volumeClaimTemplates:
  - metadata:
      name: jetstream
    spec:
      accessModes: [ReadWriteOnce]
      resources:
        requests:
          storage: 100Gi
      storageClassName: gp3-encrypted
```

### PostgreSQL HA

```yaml
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: postgres
  namespace: exchange-database
spec:
  instances: 2
  primaryUpdateStrategy: unsupervised
  postgresql:
    parameters:
      max_connections: "500"
      shared_buffers: "8GB"
      effective_cache_size: "24GB"
      maintenance_work_mem: "2GB"
      checkpoint_completion_target: "0.9"
      wal_buffers: "16MB"
      default_statistics_target: "100"
      random_page_cost: "1.1"
      effective_io_concurrency: "200"
      work_mem: "4194kB"
      min_wal_size: "2GB"
      max_wal_size: "8GB"
  bootstrap:
    initdb:
      database: exchange
      owner: exchange
      secret:
        name: postgres-credentials
  storage:
    size: 500Gi
    storageClass: gp3-encrypted
    walStorage:
      size: 100Gi
      storageClass: gp3-encrypted
  monitoring:
    enabled: true
  resources:
    requests:
      cpu: "4"
      memory: "16Gi"
    limits:
      cpu: "8"
      memory: "32Gi"
```

### Redis Cluster

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: redis-config
  namespace: exchange-infra
data:
  redis.conf: |
    cluster-enabled yes
    cluster-config-file nodes.conf
    cluster-node-timeout 5000
    save 900 1
    save 300 10
    save 60 10000
    maxmemory 4gb
    maxmemory-policy allkeys-lru
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: redis
  namespace: exchange-infra
spec:
  serviceName: redis
  replicas: 3
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
      - name: redis
        image: redis:7-alpine
        command:
        - redis-server
        - /etc/redis/redis.conf
        ports:
        - containerPort: 6379
        - containerPort: 16379
        resources:
          requests:
            cpu: "1"
            memory: "4Gi"
          limits:
            cpu: "2"
            memory: "8Gi"
        volumeMounts:
        - name: config
          mountPath: /etc/redis
        - name: data
          mountPath: /data
      volumes:
      - name: config
        configMap:
          name: redis-config
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: [ReadWriteOnce]
      resources:
        requests:
          storage: 50Gi
      storageClassName: gp3-encrypted
```

## Network Topology

### Service Mesh (Istio)

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: exchange
spec:
  hosts:
  - "*"
  gateways:
  - exchange-gateway
  http:
  - match:
    - uri:
        prefix: /api/v1/
    route:
    - destination:
        host: gateway
        port:
          number: 8080
    timeout: 30s
    retries:
      attempts: 3
      perTryTimeout: 10s
---
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
spec:
  mtls:
    mode: STRICT
```

### Network Policies

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: engine-policy
  namespace: exchange-engine
spec:
  podSelector:
    matchLabels:
      app: engine
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: exchange-gateway
    ports:
    - protocol: TCP
      port: 5000
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          name: exchange-infra
    ports:
    - protocol: TCP
      port: 4222  # NATS
  - to:
    - namespaceSelector:
        matchLabels:
          name: exchange-database
    ports:
    - protocol: TCP
      port: 5432  # PostgreSQL
```

## Capacity Planning

### Traffic Estimates

| Metric | Value | Notes |
|--------|-------|-------|
| Active users | 50,000 | Concurrent |
| Orders/sec (peak) | 100,000 | Across all symbols |
| Trades/sec (peak) | 10,000 | 10% fill rate |
| WebSocket connections | 50,000 | Market data subscribers |
| API requests/sec | 500,000 | Including reads |

### Resource Requirements

| Component | CPU | RAM | Storage | Nodes |
|-----------|-----|-----|---------|-------|
| API Gateway (3x) | 2c each | 4Gi each | - | 3 |
| Engine Shards (10x) | 4c each | 8Gi each | 100Gi each | 10 |
| Balance Service (3x) | 2c each | 4Gi each | - | 3 |
| Settlement (3x) | 2c each | 4Gi each | - | 3 |
| Market Data (3x) | 2c each | 8Gi each | - | 3 |
| NATS (3x) | 2-4c each | 4-8Gi each | 100Gi each | 3 |
| PostgreSQL (2x) | 4-8c each | 16-32Gi each | 500Gi each | 2 |
| Redis (3x) | 1-2c each | 4-8Gi each | 50Gi each | 3 |
| Observability | 8c | 32Gi | 500Gi | 1 |
| **Total** | ~100 cores | ~400Gi | ~3TB | ~30 |

### Instance Types (AWS)

| Role | Instance Type | CPU | RAM | Network |
|------|---------------|-----|-----|---------|
| Gateway | c5.xlarge | 4c | 8Gi | 10 Gbps |
| Engine | c5.xlarge | 4c | 8Gi | 10 Gbps |
| Database | r5.2xlarge | 8c | 64Gi | 10 Gbps |
| NATS | c5.2xlarge | 8c | 16Gi | 10 Gbps |
| Redis | r5.large | 2c | 16Gi | 10 Gbps |

## Deployment Strategy

### Rolling Updates

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: gateway
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0
      maxSurge: 1
```

### Canary Deployment

```yaml
apiVersion: flagger.app/v1beta1
kind: Canary
metadata:
  name: engine
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: engine
  service:
    port: 5000
  analysis:
    interval: 1m
    threshold: 5
    maxWeight: 50
    stepWeight: 10
    metrics:
    - name: request-success-rate
      thresholdRange:
        min: 99
    - name: request-duration
      thresholdRange:
        max: 500
  webhooks:
  - name: load-test
    url: http://flagger-loadtester/
    timeout: 5s
    metadata:
      cmd: "hey -z 1m -q 10 -c 2 http://engine-canary:5000/health"
```

## Disaster Recovery

### Backup Strategy

```yaml
apiVersion: velero.io/v1
kind: Schedule
metadata:
  name: daily-backup
  namespace: velero
spec:
  schedule: "0 2 * * *"  # 2 AM daily
  template:
    includedNamespaces:
    - exchange-engine
    - exchange-backend
    - exchange-database
    storageLocation: aws-backups
    volumeSnapshotLocations:
    - aws-snapshots
    ttl: 720h  # 30 days
```

### Snapshot Strategy

```bash
#!/bin/bash
# Snapshot script (cron)

# 1. Snapshot engine order books
kubectl exec -n exchange-engine engine-shard-0 -- \
  curl -X POST http://localhost:5000/admin/snapshot

# 2. Backup PostgreSQL
kubectl exec -n exchange-database postgres-0 -- \
  pg_dumpall | gzip > /backup/postgres-$(date +%Y%m%d).sql.gz

# 3. Upload to S3
aws s3 sync /backup/ s3://exchange-backups/$(date +%Y%m%d)/
```

### Recovery Procedure

```bash
#!/bin/bash
# Recovery procedure

# 1. Restore from Velero backup
velero restore create --from-backup daily-backup-20250114

# 2. If needed, restore engine snapshots
for shard in 0 1 2 3 4; do
  kubectl cp snapshots/engine-shard-${shard}-latest.bin \
    exchange-engine/engine-shard-${shard}:/data/snapshots/latest.bin
  kubectl exec -n exchange-engine engine-shard-${shard} -- \
    curl -X POST http://localhost:5000/admin/recover
done
```

## Configuration Management

### ConfigMaps

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: engine-config
  namespace: exchange-engine
data:
  config.toml: |
    [sharding]
    enabled = true
    symbols_per_shard = 10

    [snapshot]
    interval_events = 1000
    interval_seconds = 30
    retain_count = 5

    [trading]
    max_order_value = 1000000
    max_price_deviation_percent = 10

    [nats]
    url = "nats://nats.exchange-infra.svc.cluster.local:4222"
    jetstream_enabled = true
```

### Secrets

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: database-credentials
type: Opaque
stringData:
  url: "postgresql://exchange:password@postgres.exchange-database.svc.cluster.local:5432/exchange"
  username: exchange
  password: password
---
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: api-secrets
spec:
  encryptedData:
    jwt-secret: AgBy3i4OJSWK+PiTySY...
    api-key-hash: AgBy3i4OJSWK+PiTySY...
```

## Monitoring & Alerting

### Prometheus ServiceMonitors

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: engine
  namespace: exchange-engine
spec:
  selector:
    matchLabels:
      app: engine
  endpoints:
  - port: metrics
    interval: 15s
    path: /metrics
```

### Alertmanager Config

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: alertmanager-config
  namespace: exchange-observability
data:
  alertmanager.yaml: |
    global:
      resolve_timeout: 5m

    route:
      receiver: 'pagerduty'
      group_by: ['alertname', 'cluster']
      group_wait: 10s
      group_interval: 5m
      repeat_interval: 12h

    receivers:
    - name: 'pagerduty'
      pagerduty_configs:
      - service_key: '<PAGERDUTY_KEY>'
```

## Scaling Procedures

### Horizontal Pod Autoscaling

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: gateway-hpa
  namespace: exchange-gateway
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: gateway
  minReplicas: 3
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Percent
        value: 50
        periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
      - type: Percent
        value: 100
        periodSeconds: 30
```

### Cluster Autoscaling

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: ClusterAutoscaler
metadata:
  name: cluster-autoscaler
spec:
  scaleDownUnneededTime: 10m
  maxNodesPerNodeGroup: 10
  nodeGroups:
  - name: engine-nodes
    minSize: 10
    maxSize: 50
```

## Operational Procedures

### Graceful Shutdown

```rust
use tokio::signal;

#[tokio::main]
async fn main() -> Result<(), Error> {
    let engine = MatchingEngine::new(config).await?;

    // Create shutdown signal
    let (shutdown_tx, mut shutdown_rx) = tokio::sync::broadcast::channel(1);

    // Handle SIGTERM
    let mut sigterm = signal::unix::signal(signal::unix::SignalKind::terminate())?;
    tokio::spawn(async move {
        let _ = sigterm.recv().await;
        warn!("SIGTERM received, initiating graceful shutdown");
        let _ = shutdown_tx.send(());
    });

    // Wait for shutdown signal
    shutdown_rx.recv().await?;

    // Create final snapshot before shutdown
    info!("Creating final snapshot");
    engine.create_snapshot().await?;

    // Close NATS connection
    info!("Closing NATS connection");
    engine.close_nats().await?;

    info!("Shutdown complete");
    Ok(())
}
```

### Blue-Green Deployment

```bash
#!/bin/bash
# Blue-green deployment script

BLUE="exchange-blue"
GREEN="exchange-green"

# Determine active color
ACTIVE=$(kubectl get svc exchange -o jsonpath='{.spec.selector.color}')

if [ "$ACTIVE" = "blue" ]; then
  TARGET="green"
else
  TARGET="blue"
fi

echo "Deploying to $TARGET environment"

# Deploy new version
kubectl apply -f manifests/$TARGET/

# Wait for rollout
kubectl rollout status deployment/gateway-$TARGET

# Run smoke tests
./scripts/smoke-test.sh $TARGET

if [ $? -eq 0 ]; then
  echo "Tests passed, switching traffic"
  kubectl patch svc exchange -p '{"spec":{"selector":{"color":"'$TARGET'"}}}'
else
  echo "Tests failed, rolling back"
  kubectl rollout undo deployment/gateway-$TARGET
  exit 1
fi
```

## Checklist

### Pre-Deployment

- [ ] All tests passing
- [ ] Security scan completed
- [ ] Performance benchmarks met
- [ ] Load tests passed
- [ ] Configuration validated
- [ ] Secrets prepared
- [ ] Database migrations prepared
- [ ] Rollback plan documented

### Post-Deployment

- [ ] Health checks passing
- [ ] Metrics normalized
- [ ] No errors in logs
- [ ] Trades flowing normally
- [ ] Latency within SLA
- [ ] Alerts firing correctly
