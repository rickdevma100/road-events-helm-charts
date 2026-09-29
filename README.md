# Road Events Helm Charts

Infrastructure and deployment Helm charts for the Road-Event Capture App & Smart Bulb system running on MicroK8s.

## Architecture & Storage
- **PostGIS**: Relational database running on Kubernetes with a 10 GiB `microk8s-hostpath` PVC for spatial tables, GiST indexes, and ride state.
- **Photos**: Stored directly in **MinIO S3 object storage** (`http://lang-learn-minio.lang-learn.svc.cluster.local:9000`), bucket `road-events`. Eliminates the need for a separate photo filesystem PVC.

## Directory Structure

```text
charts/
├── postgres-postgis/       # PostGIS 15 + btree_gist database deployment with hostpath PVC
│   ├── Chart.yaml
│   ├── values.yaml
│   └── templates/
│       ├── namespace.yaml
│       ├── secret.yaml
│       ├── pvc.yaml
│       ├── deployment.yaml
│       └── service.yaml
├── road-events-api/        # FastAPI backend, MinIO S3 object storage config, and ngrok tunnel
│   ├── Chart.yaml
│   ├── values.yaml
│   └── templates/
│       ├── secret.yaml
│       ├── deployment.yaml
│       ├── service.yaml
│       └── ngrok-tunnel.yaml
└── road-events/            # Umbrella chart managing the full stack
    ├── Chart.yaml
    └── values.yaml
```

## Quick Start on MicroK8s

### 1. Deploy PostGIS Database
```bash
microk8s helm upgrade --install postgres-postgis ./charts/postgres-postgis \
  --namespace road-events \
  --create-namespace
```

### 2. Deploy FastAPI Backend
```bash
microk8s helm upgrade --install road-events-api ./charts/road-events-api \
  --namespace road-events \
  --set config.bulb.adapter="tuya" \
  --set config.deviceToken="your-secure-device-token"
```

### 3. Deploy Umbrella Stack (All-in-One)
```bash
cd charts/road-events
microk8s helm dependency build
microk8s helm upgrade --install road-events . \
  --namespace road-events \
  --create-namespace
```

## Configuration Reference

### PostGIS Chart (`charts/postgres-postgis`)
| Parameter | Description | Default |
|---|---|---|
| `namespace.name` | Kubernetes namespace | `road-events` |
| `image.repository` | ARM64 & AMD64 compatible image | `kartoza/postgis` |
| `image.tag` | PostGIS version | `15-3.3` |
| `auth.database` | Database name | `roadevents` |
| `auth.username` | Database user | `postgres` |
| `auth.password` | Database password | `postgres_password_123` |
| `persistence.claimName` | PVC name | `postgres-postgis-pvc` |
| `persistence.storageClass` | Storage class | `microk8s-hostpath` |
| `persistence.size` | PVC volume size | `10Gi` |

### API Chart (`charts/road-events-api`)
| Parameter | Description | Default |
|---|---|---|
| `replicaCount` | Single-replica constraint | `1` |
| `config.deviceToken` | Bearer token for client auth | `secret-device-token-12345` |
| `config.bulb.adapter` | Bulb adapter (`mock` or `tuya`) | `mock` |
| `config.bulb.ip` | Home bulb local IP | `192.168.0.109` |
| `config.bulb.deviceId` | Tuya bulb device ID | `d798b17a8abfe1e652uoe1` |
| `config.bulb.localKey` | 16-character local key | `t5lS>p7jES(}Lby}` |
| `config.bulb.protocol` | Tuya protocol version | `3.5` |
| `s3.endpointUrl` | MinIO internal S3 URL | `http://lang-learn-minio.lang-learn.svc.cluster.local:9000` |
| `s3.bucketName` | S3 photo bucket | `road-events` |
| `s3.accessKey` | MinIO access key | `minioadmin` |
| `s3.secretKey` | MinIO secret key | `minioadmin` |
| `ngrok.enabled` | Ingress tunnel agent | `false` |