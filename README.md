# Kubernetes Observability Stack

A production-grade observability platform for Kubernetes running on AWS EKS and on-premises clusters. Covers logs, metrics, and traces using fully open-source components.

## Architecture Overview

```text
                  ┌──────────────────────────────┐
                  │        SPOKE CLUSTERS        │
                  │  (EKS / On-Prem / OpenShift) │
                  │                              │
                  │  ┌──────────────────────┐    │
                  │  │     OTel Agent       │    │
                  │  │     (DaemonSet)      │    │
                  │  └──────────┬───────────┘    │
                  └─────────────┼────────────────┘
                                │ mTLS + Bearer token
                                │ OTLP/gRPC
                                ▼
               ┌──────────────────────────────────────┐
               │          HUB CLUSTER                 │
               │                                      │
               │  ┌─────────────────┐                 │
               │  │   OTel Gateway  │                 │
               │  │   (Deployment)  │                 │
               │  └──┬──────┬───────┘                 │
               │     │      │    │                     │
               │  Logs│  Metrics│  Traces              │
               │     ▼      ▼    ▼                     │
               │  ┌────┐ ┌─────┐ ┌───────┐            │
               │  │Loki│ │Mimir│ │ Tempo │            │
               │  └────┘ └─────┘ └───────┘            │
               │                                      │
               │  ┌─────────┐                         │
               │  │ Grafana │                         │
               │  │  (UI)   │                         │
               │  └─────────┘                         │
               └──────────────────────────────────────┘
```

### Signal routing

| Signal  | Spoke → Hub                  | Storage        |
|---------|------------------------------|----------------|
| Logs    | OTel Agent → Gateway → Loki  | AWS S3 / MinIO |
| Metrics | OTel Agent → Gateway → Mimir | AWS S3 / MinIO |
| Traces  | OTel Agent → Gateway → Tempo | AWS S3 / MinIO |

---

## Repository Layout

```text
.
├── spoke/                  # Spoke-cluster manifests (agent side)
│   ├── agent.yml           # OTel Collector DaemonSet (OTel Operator CRD)
│   ├── token.yml           # Bearer-token secret for spoke→hub auth
│   └── instrumentation.yml # Auto-instrumentation for Java workloads
│
├── hub/                    # Hub-cluster manifests (aggregation side)
│   ├── gateway.yml         # OTel Collector Deployment (gateway mode)
│   ├── token.yml           # Bearer-token secret validated by the gateway
│   ├── ingress.yml         # NGINX Ingress (Kubernetes) for the gateway
│   └── route.yml           # OpenShift Route alternative for the gateway
│
├── tools/                  # Hub-cluster Helm values for storage backends
│   ├── loki-logs.yml       # Grafana Loki (SimpleScalable, S3 backend)
│   ├── mimir-fs.yml        # Grafana Mimir (MinIO backend — on-prem)
│   ├── tempo-fs.yml        # Grafana Tempo Distributed (MinIO backend)
│   ├── pyroscope-fs.yml    # Grafana Pyroscope Distributed (MinIO backend)
│   ├── grafana.yml         # Grafana Operator CR (Azure AD SSO)
│   ├── prometheus.yaml     # Prometheus CR (remote-write to Mimir)
│   ├── s3-helm/            # Helm values for AWS S3-backed deployments
│   │   ├── mimir-metrics.yml   # Mimir with real AWS S3
│   │   └── traces-tempo.yml    # Tempo with real AWS S3
│   └── s3-lifecycle-policy/    # AWS S3 lifecycle rules for retention
│       ├── loki.json
│       ├── mimir.json
│       ├── tempo.json
│       └── cmds.txt
│
├── db-exporter/            # Database metrics exporters
│   ├── yace-values.yml     # YACE — AWS CloudWatch/RDS exporter (IRSA auth)
│   ├── azure-metrics-exporter.yml  # Azure PostgreSQL Flexible Server exporter
│   └── azure-metrics-sm.yml        # ServiceMonitor for Azure exporter
│
└── servicemonitor/         # Prometheus ServiceMonitor CRDs
    ├── cadvisor.yml        # cAdvisor / kubelet metrics
    ├── ksm.yml             # kube-state-metrics
    └── kube-state-metrics-ksm.yml
```

---

## Prerequisites

| Requirement               | Notes                                        |
|---------------------------|----------------------------------------------|
| Kubernetes 1.27+          | EKS, GKE, on-prem, or OpenShift              |
| Helm 3.x                  | For all chart installs                       |
| OTel Operator             | Manages `OpenTelemetryCollector` CRDs        |
| Prometheus Operator       | Manages `ServiceMonitor` / `PodMonitor` CRDs |
| cert-manager              | Issues mTLS certs for spoke↔hub              |
| NGINX Ingress Controller  | For Kubernetes-based hub ingress             |
| AWS EKS Pod Identity/IRSA | For S3 and CloudWatch access (EKS spokes)    |

---

## Quick Start

### 1. Hub — Deploy storage backends

```bash
# Loki (logs)
helm upgrade --install loki grafana/loki \
  -n loki --create-namespace \
  -f tools/loki-logs.yml

# Mimir (metrics) — MinIO backend
helm upgrade --install mimir grafana/mimir-distributed \
  -n mimir-ms --create-namespace \
  -f tools/mimir-fs.yml

# Tempo (traces) — MinIO backend
helm upgrade --install tempo grafana/tempo-distributed \
  -n tempo-ms --create-namespace \
  -f tools/tempo-fs.yml

# Pyroscope (profiles) — MinIO backend
helm upgrade --install pyroscope grafana/pyroscope \
  -n pyroscope-ms --create-namespace \
  -f tools/pyroscope-fs.yml

# Grafana (UI)
kubectl apply -f tools/grafana.yml
```

### 2. Hub — Deploy the OTel Gateway

```bash
# Create the auth token secret first
kubectl apply -f hub/token.yml

# Deploy the gateway
kubectl apply -f hub/gateway.yml

# Expose the gateway (choose one)
kubectl apply -f hub/ingress.yml    # Kubernetes / NGINX
kubectl apply -f hub/route.yml      # OpenShift
```

### 3. Spoke — Deploy the OTel Agent

```bash
# Create the same auth token on the spoke
kubectl apply -f spoke/token.yml

# Deploy the agent DaemonSet
kubectl apply -f spoke/agent.yml

# (Optional) Auto-instrumentation for Java apps
kubectl apply -f spoke/instrumentation.yml
```

---

## Configuration

### Replacing placeholders

All sensitive values have been replaced with placeholders. Search for these strings and substitute your real values before deploying:

| Placeholder                                  | What to replace with                        |
|----------------------------------------------|---------------------------------------------|
| `example.com`                                | Your actual domain (e.g. `acme.com`)        |
| `YOUR-CLUSTER-NAME`                          | Your Kubernetes cluster name                |
| `111122223333`                               | Your AWS account ID (spoke cluster)         |
| `444455556666`                               | Your AWS account ID (hub cluster)           |
| `your-account-id`                            | Your AWS account ID in S3 bucket names      |
| `your-aws-profile`                           | Your AWS CLI profile name                   |
| `your-rds-instance`                          | Your RDS instance identifier                |
| `change-me-minio-password`                   | A strong random password for MinIO          |
| `00000000-0000-0000-0000-000000000001`       | Azure Workload Identity client ID           |
| `00000000-0000-0000-0000-000000000002`       | Azure tenant ID                             |
| `00000000-0000-0000-0000-000000000003`       | Azure subscription ID                       |
| `00000000-0000-0000-0000-000000000010/11/12` | Azure AD group IDs (Admin/Viewer/Editor)    |

### mTLS Setup

The spoke agents authenticate to the hub gateway using mutual TLS + bearer token:

1. **Issue a CA** with cert-manager (`ClusterIssuer` or `Issuer`).
2. **Hub**: create `otel-gateway-tls` (server cert) and `otel-gateway-ca` (CA) secrets in `opentelemetry` namespace.
3. **Spoke**: create `otel-agent-tls` (client cert) and `otel-agent-ca` (CA) secrets in `opentelemetry` namespace.
4. **Token**: generate a random bearer token, set it in `hub/token.yml` and `spoke/token.yml` as `OTEL_AUTH_TOKEN`.

### AWS IAM (IRSA / Pod Identity)

For EKS spokes with CloudWatch or S3 access:

```bash
# Create IAM policy from the provided template
aws iam create-policy \
  --policy-name otel-agent-cloudwatch-policy \
  --policy-document file://db-exporter/cw-policy.json \
  --profile your-aws-profile

# Attach to service account via eksctl
eksctl create iamserviceaccount \
  --name otel-agent-collector \
  --namespace opentelemetry \
  --cluster <your-cluster-name> \
  --region ap-southeast-1 \
  --attach-policy-arn arn:aws:iam::<account-id>:policy/otel-agent-cloudwatch-policy \
  --override-existing-serviceaccounts \
  --approve \
  --profile your-aws-profile
```

### S3 Lifecycle Policies

Apply retention policies to your S3 buckets:

```bash
aws s3api put-bucket-lifecycle-configuration \
  --bucket otel-logs-your-account-id-ap-southeast-2 \
  --lifecycle-configuration file://tools/s3-lifecycle-policy/loki.json \
  --profile your-aws-profile
```

---

## Component Versions

| Component              | Version |
|------------------------|---------|
| OTel Collector Contrib | 0.149.0 |
| Grafana Loki           | 3.5.7   |
| Grafana Pyroscope      | 1.18.0  |

---

## Key Design Decisions

- **mTLS + Bearer token** — spoke agents use mutual TLS certificates AND a bearer token to authenticate with the hub gateway. Belt and suspenders.
- **Compression everywhere** — `zstd` on all OTLP exporters. Reduces cross-cluster bandwidth significantly.
- **Target Allocator** — the OTel Operator's Target Allocator scrapes `ServiceMonitor`/`PodMonitor` CRDs directly, replacing a separate Prometheus instance on spoke clusters.
- **Per-signal batching** — separate `batch` processors for logs, metrics, and traces with different tuning (batch size and timeout) matched to each signal's volume and latency requirements.
- **MinIO for on-prem** — on-prem clusters use in-cluster MinIO as an S3-compatible backend. EKS clusters use real AWS S3 with IRSA/Pod Identity — no credentials needed.
- **Attribute cleanup** — the `resource/cleanup` processor strips noisy SDK-injected attributes (process info, SDK versions) before forwarding to reduce storage cost.

---

## Troubleshooting

### Agent can't connect to gateway

- Verify mTLS certs are not expired: `kubectl get cert -n opentelemetry`
- Check bearer token matches on both sides
- Confirm DNS resolves: `nslookup otel-gateway.example.com`

### High memory usage on agent

- Tune `memory_limiter` in `spoke/agent.yml` — `limit_mib` should be ~75–80% of the pod memory limit
- Reduce `queue_size` on exporters if the queue is filling up

### Loki / Mimir returning 429 (rate limit)

- Increase `ingestion_rate_mb` / `ingestion_burst_size_mb` in `tools/loki-logs.yml`
- For Mimir, adjust `ingestion_rate` / `ingestion_burst_size`

---

## License

MIT
