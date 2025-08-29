# OpenTelemetry Observability Stack

This repository contains a comprehensive observability stack for Kubernetes environments, featuring OpenTelemetry, Prometheus, Grafana, Jaeger, and Loki with Fluent Bit for log aggregation.

- **OpenTelemetry Collector**: Collects, processes, and exports telemetry data (metrics, traces, logs)
- **Prometheus**: Metrics collection and storage
- **Grafana**: Visualization and dashboards with pre-configured datasources
- **Jaeger**: Distributed tracing with Elasticsearch backend
- **Loki**: Log aggregation and storage with S3 backend
- **Fluent Bit**: Log collection and forwarding to Loki
- **OAuth Proxy**: Authentication layer for secure access to observability tools

## Architecture Overview

This repository implements a **hub-and-spoke architecture** for distributed observability:

- **Hub**: Central observability infrastructure that aggregates, stores, and visualizes telemetry data
- **Spoke**: Distributed collection agents deployed across environments that forward data to the hub

## Directory Structure

```
├── hub/                           # Hub: Central observability infrastructure
│   ├── kustomization.yaml         # Hub aggregation configuration
│   ├── loadlbalancer/             # Load balancers for hub services
│   │   ├── jaeger-lb.yml          # Jaeger load balancer
│   │   ├── loki-lb.yml            # Loki load balancer
│   │   ├── prometheus-lb.yml      # Prometheus load balancer
│   │   └── kustomization.yaml     # Load balancer aggregation
│   ├── logs/                      # Centralized log storage (Loki)
│   │   ├── loki-values.yml        # Loki Helm values
│   │   ├── secret.yml             # S3 credentials for Loki
│   │   ├── policy.json            # S3 bucket policy
│   │   └── command.txt            # Deployment commands
│   ├── metrics/                   # Centralized metrics storage (Thanos)
│   │   ├── thanos-values.yaml     # Thanos Helm values
│   │   ├── objstore.yaml          # Object storage configuration
│   │   ├── policy.json            # S3 bucket policy
│   │   └── commands.txt           # Deployment commands
│   ├── traces/                    # Centralized trace storage (Jaeger)
│   │   ├── jaeger-hub.yml         # Jaeger hub configuration
│   │   ├── elasticsearch.yml      # Elasticsearch backend
│   │   ├── ilm.txt                # Index lifecycle management
│   │   └── kustomization.yaml     # Tracing aggregation
│   └── visual/                    # Visualization and dashboards
│       ├── grafana.yml            # Grafana configuration
│       ├── jaeger-ds.yml          # Jaeger datasource
│       ├── loki-ds.yml            # Loki datasource
│       ├── prometheus-ds.yml      # Prometheus datasource
│       └── kustomization.yaml     # Visualization aggregation
├── spoke/                         # Spoke: Distributed collection agents
│   ├── kustomization.yaml         # Spoke aggregation configuration
│   ├── logs/                      # Log collection (Fluent Bit)
│   │   ├── fluentbit.yml          # Fluent Bit DaemonSet
│   │   ├── commands.txt           # Deployment commands
│   │   └── kustomization.yaml     # Log collection aggregation
│   ├── metrics/                   # Metrics collection (Prometheus)
│   │   ├── prometheus.yaml        # Prometheus configuration
│   │   ├── servicemonitor.yml     # Service monitoring rules
│   │   ├── objstore.yaml          # Object storage for remote write
│   │   └── kustomization.yaml     # Metrics collection aggregation
│   └── opentelemetry/             # OpenTelemetry collection
│       ├── collector.yml          # OpenTelemetry Collector DaemonSet
│       ├── instrumentation.yml    # Auto-instrumentation configuration
│       └── kustomization.yaml     # OpenTelemetry aggregation
├── validator.sh                   # Validation script
└── .gitignore                     # Git ignore rules
```

## Container Images

This observability stack uses several container images from different registries. Here's a comprehensive list of the images used:

### Hub Components

| Component | Image | Registry | Current Version |
|-----------|-------|----------|-----------------|
| **Grafana** | `docker.io/grafana/grafana` | Docker Hub | `@sha256:fe89b739a264c78f2111d68221a1d51db67135ec50885dc93b59a981a7a5d4d5` |
| **Jaeger Hub** | `cr.jaegertracing.io/jaegertracing/jaeger` | Jaeger Registry | `2.9.0` |
| **Loki** | `docker.io/grafana/loki` | Docker Hub | Via Helm Chart |
| **Thanos (Hub)** | `quay.io/thanos/thanos` | Quay.io | Via Helm Chart |

### Spoke Components

| Component | Image | Registry | Current Version |
|-----------|-------|----------|-----------------|
| **OpenTelemetry Collector** | `otel/opentelemetry-collector-contrib` | Docker Hub | `0.133.0` |
| **Fluent Bit** | `cr.fluentbit.io/fluent/fluent-bit` | Fluent Bit Registry | Via Helm Chart |
| **Prometheus** | `quay.io/prometheus/prometheus` | Quay.io | Via Operator |
| **Thanos Sidecar** | `quay.io/thanos/thanos` | Quay.io | `v0.39.2` |

### Image Update Strategy

#### Checking for Latest Images

**1. OpenTelemetry Collector:**
- **Registry**: [Docker Hub - OpenTelemetry Collector Contrib](https://hub.docker.com/r/otel/opentelemetry-collector-contrib/tags)
- **GitHub Releases**: [OpenTelemetry Collector Releases](https://github.com/open-telemetry/opentelemetry-collector-contrib/releases)
- **Update Location**: `spoke/opentelemetry/collector.yml` - Update the `image` field

**2. Jaeger:**
- **Registry**: [Jaeger Container Registry](https://www.jaegertracing.io/docs/1.60/deployment/#container-images)
- **GitHub Releases**: [Jaeger Releases](https://github.com/jaegertracing/jaeger/releases)
- **Update Location**: `hub/traces/jaeger-hub.yml` - Update the `image` field

**3. Grafana:**
- **Registry**: [Docker Hub - Grafana](https://hub.docker.com/r/grafana/grafana/tags)
- **GitHub Releases**: [Grafana Releases](https://github.com/grafana/grafana/releases)
- **Update Location**: `hub/visual/grafana.yml` - Update the `version` field

**4. Thanos:**
- **Registry**: [Quay.io - Thanos](https://quay.io/repository/thanos/thanos?tab=tags)
- **GitHub Releases**: [Thanos Releases](https://github.com/thanos-io/thanos/releases)
- **Update Locations**: 
  - `spoke/metrics/prometheus.yaml` - Update the `thanos.image` and `thanos.version` fields
  - `hub/metrics/thanos-values.yaml` - Update Helm values

**5. Helm Chart Managed Images:**
- **Loki**: Check [Grafana Helm Charts](https://github.com/grafana/helm-charts/tree/main/charts/loki-distributed)
- **Fluent Bit**: Check [Fluent Bit Helm Charts](https://github.com/fluent/helm-charts/tree/main/charts/fluent-bit)

**For Helm Chart Managed Images:**

```bash
# Update Loki
helm repo update
helm upgrade loki grafana/loki-distributed \
  --namespace opentelemetry \
  --values hub/logs/loki-values.yml

# Update Fluent Bit  
helm repo update
helm upgrade fluentbit fluent/fluent-bit \
  --namespace opentelemetry \
  --values spoke/logs/fluentbit.yml

# Update Thanos (if using Helm)
helm repo update
helm upgrade thanos bitnami/thanos \
  --namespace opentelemetry \
  --values hub/metrics/thanos-values.yaml
```

## Prerequisites

- Kubernetes cluster (OpenShift compatible)
- Helm 3.x
- kubectl or oc CLI configured to access your cluster
- S3-compatible storage for Loki (AWS S3, MinIO, etc.)
- Appropriate storage classes configured (e.g., `gp3-csi` for AWS)
- For OpenShift: Cluster admin privileges to manage Security Context Constraints (SCCs)

## Quick Start

### 1. Deploy Hub Infrastructure

Deploy the central observability hub that provides storage, visualization, and load balancing:

```bash
# Deploy the complete hub infrastructure
kubectl apply -k hub/

# Or deploy hub components individually
kubectl apply -k hub/loadlbalancer/    # Load balancers for hub services
kubectl apply -k hub/traces/           # Jaeger tracing with Elasticsearch
kubectl apply -k hub/visual/           # Grafana with datasources
```

### 2. Deploy Spoke Collection Agents

Deploy the distributed collection agents that forward telemetry data to the hub:

```bash
# Deploy all spoke components
kubectl apply -k spoke/

# Or deploy spoke components individually
kubectl apply -k spoke/opentelemetry/  # OpenTelemetry collectors
kubectl apply -k spoke/logs/           # Fluent Bit log collection
kubectl apply -k spoke/metrics/        # Prometheus metrics collection
```

### 3. Hub-and-Spoke Deployment Pattern

The architecture supports flexible deployment patterns:

**Centralized Hub Deployment:**
- Deploy hub components in a central cluster or namespace
- Hub provides shared storage backends (Loki, Jaeger, Thanos)
- Hub includes visualization layer (Grafana) and load balancers

**Distributed Spoke Deployment:**
- Deploy spoke components across multiple environments/clusters
- Spokes collect and forward telemetry data to the central hub
- Each spoke can be configured independently for different environments

**Hybrid Deployment:**
- Hub and spoke components can be deployed in the same cluster for single-cluster observability
- Or distributed across multiple clusters for multi-cluster observability

**Labeling Strategy:**
All resources are labeled with consistent Kubernetes-standard labels for better organization and management:
- `app.kubernetes.io/part-of: observability-stack` - Applied to all resources
- `app.kubernetes.io/managed-by: kustomize` - Applied to all resources
- Component-specific labels:
  - Jaeger: `app.kubernetes.io/name: jaeger`, `app.kubernetes.io/component: tracing`
  - OpenTelemetry: `app.kubernetes.io/name: opentelemetry`, `app.kubernetes.io/component: collector`
  - Prometheus: `app.kubernetes.io/name: prometheus`, `app.kubernetes.io/component: monitoring`
  - Grafana: `app.kubernetes.io/name: grafana`, `app.kubernetes.io/component: visualization`
  - OAuth Proxy: `app.kubernetes.io/name: oauth-proxy-{service}`, `app.kubernetes.io/component: security`

### 4. Deploy Hub Storage Components with Helm

Some hub components require Helm deployment for advanced configurations:

#### Deploy Loki (Hub Log Storage)

```bash
# Add Grafana Helm repository
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

# Create namespace
kubectl create namespace opentelemetry

# Configure S3 credentials (update the secret with your values)
kubectl apply -f hub/logs/secret.yml -n opentelemetry

# Deploy Loki with custom values
helm upgrade --install loki grafana/loki-distributed \
  --namespace opentelemetry \
  --values hub/logs/loki-values.yml

# For OpenShift, add required security context constraints
oc adm policy add-scc-to-user anyuid -z loki-loki-distributed
```

#### Deploy Thanos (Hub Metrics Storage)

```bash
# Add Bitnami Helm repository
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

# Configure object storage credentials
kubectl apply -f hub/metrics/objstore.yaml -n opentelemetry

# Deploy Thanos with custom values
helm upgrade --install thanos bitnami/thanos \
  --namespace opentelemetry \
  --values hub/metrics/thanos-values.yaml
```

## Configuration Details

### Loki Configuration

The Loki setup includes:
- **Distributed architecture** with separate components (distributor, ingester, querier, compactor)
- **S3 storage backend** for long-term log retention
- **720-hour retention period** (30 days)
- **Persistent storage** for ingester and compactor components
- **Memberlist-based service discovery**

Key features:
- Horizontal scaling support
- S3-compatible object storage
- Built-in log retention and compaction
- Query optimization with caching

### Fluent Bit Configuration

The Fluent Bit setup provides:
- **DaemonSet deployment** for log collection from all nodes
- **Kubernetes metadata enrichment** with pod, namespace, and container information
- **Namespace filtering** (focuses on `qam` and `load-testing` namespaces)
- **OpenTelemetry trace correlation** with trace_id and span_id extraction
- **JSON log parsing** for structured logging
- **Loki output** with proper labeling for efficient querying
- **Storage configuration** with persistent buffer and checkpoint files
- **Resource limits** optimized for container environments

### OpenTelemetry Collector

Configured as a DaemonSet with:
- **OTLP receivers** (gRPC and HTTP)
- **Jaeger receivers** (gRPC, Thrift Compact, Thrift HTTP)
- **Host metrics collection** (CPU, memory, network, load)
- **Kubernetes attributes processor** for resource enrichment
- **Prometheus metrics export**
- **Zipkin trace export** to Jaeger

### Grafana Datasources

Pre-configured datasources for:
- **Prometheus**: Metrics visualization
- **Jaeger**: Distributed tracing
- **Loki**: Log exploration and correlation

## Security Configuration

### S3 Credentials

Before deploying Loki, update the S3 credentials in `hub/logs/secret.yml`:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: grafana-loki-s3-secret
data:
  ACCESS_KEY_ID: <base64-encoded-access-key>
  ACCESS_KEY_SECRET: <base64-encoded-secret-key>
  BUCKETNAME: <base64-encoded-bucket-name>
  ENDPOINT: <base64-encoded-s3-endpoint>
  REGION: <base64-encoded-region>
type: Opaque
```

## S3 Storage and Lifecycle Management

This observability stack uses S3-compatible storage for both Loki (logs) and Prometheus/Thanos (metrics) with optimized lifecycle policies to reduce storage costs while maintaining query performance.

### S3 Lifecycle Policies Overview

The lifecycle policies are designed to automatically transition data to cheaper storage tiers without breaking queries. **Important**: Loki and Thanos remain responsible for data deletion based on their retention settings; S3 rules primarily handle storage-class transitions.

### Loki S3 Bucket Lifecycle

**Configured Rules:**

1. **`loki-index-warm`** (Index optimization)
   - **Scope**: `index/` prefix
   - **Transitions**: → Intelligent-Tiering at **14 days**
   - **Rationale**: Keeps indexes accessible while reducing costs for older index data

2. **`loki-data-warm-cold`** (Chunk data optimization)
   - **Scope**: Entire bucket
   - **Transitions**: 
     - → Intelligent-Tiering at **30 days**
     - → Glacier Instant Retrieval at **180 days**
   - **Rationale**: Chunks age to cheaper tiers but remain instantly readable for queries

**Key Points:**
- Loki's compactor handles data deletion based on retention settings (30 days default)
- S3 does not expire objects - deletion is managed by Loki
- Indexes remain hot/warm to avoid Glacier restore delays
- All data remains instantly queryable regardless of storage class

### Prometheus/Thanos S3 Bucket Lifecycle

**Configured Rule:**

- **`prometheus-90day-retention`**
  - **Scope**: Entire bucket
  - **Transitions**:
    - → Standard-IA at **30 days**
    - → Glacier Instant Retrieval at **60 days**
  - **Expiration**: Objects deleted at **100 days**
  - **Noncurrent versions**: Deleted after **1 day** (if versioning enabled)

**Critical Configuration Requirements:**

⚠️ **Important**: S3 expiration at 100 days is only safe if Thanos Compactor retention ≤ 100 days for all resolutions.

**Recommended Thanos retention alignment:**
```bash
--retention.resolution-raw=30d
--retention.resolution-5m=60d  
--retention.resolution-1h=100d
```

**Safety considerations:**
- Ensure Thanos compactor is running with appropriate `--delete-delay` (default ~48h)
- If you need data beyond 100 days, disable S3 expiration and let Thanos handle deletion
- Monitor for 404/NotFound errors in Thanos logs indicating missing objects

### S3 Bucket Configuration

For both Loki and Prometheus/Thanos buckets, ensure:

```bash
# Common lifecycle settings for both buckets
- Abort incomplete multipart uploads after 7 days
- Enable appropriate storage class transitions
- Configure bucket policies for service account access
```

### Verification and Monitoring

**Regular checks:**
1. **S3 Console**: Verify storage class transitions are occurring as expected
2. **Loki queries**: Ensure older log queries work (indexes in Intelligent-Tiering, chunks in appropriate tiers)
3. **Thanos/Prometheus queries**: Verify data < 100 days is accessible, older data is properly cleaned up
4. **Application logs**: Monitor for S3-related errors in Loki and Thanos components

**Troubleshooting S3 issues:**
```bash
# Check Loki logs for S3 connectivity
kubectl logs -l app.kubernetes.io/name=loki -n opentelemetry | grep -i s3

# Check Thanos logs for object store issues  
kubectl logs -l app.kubernetes.io/name=thanos -n opentelemetry | grep -i "object store"
```

### Cost Optimization Benefits

The lifecycle policies provide significant cost savings:
- **Loki**: ~60-80% cost reduction for older log data while maintaining instant access
- **Prometheus/Thanos**: ~70-85% cost reduction with automatic cleanup preventing unbounded growth
- **Operational**: Reduced manual intervention for storage management

### Modifying Lifecycle Policies

**To adjust lifecycle rules:**

1. **Via AWS Console**: Navigate to S3 bucket → Management → Lifecycle rules
2. **Via CLI**: Apply updated lifecycle policy JSON
3. **Important**: When changing Thanos bucket expiration, first align compactor retention flags

**Change control process:**
- Test changes in non-production environment first
- Ensure Thanos/Loki retention settings are compatible
- Monitor query performance after changes
- Document any custom retention requirements

### OAuth Proxy

The repository includes OAuth proxy configurations for secure access to:
- Grafana dashboards
- Prometheus UI
- Loki query interface
- Jaeger UI

## Monitoring and Observability

### Metrics Collection

- **Application metrics**: Collected via OpenTelemetry instrumentation
- **Infrastructure metrics**: Host metrics from OpenTelemetry collector
- **Kubernetes metrics**: Service monitors and pod metrics

### Distributed Tracing

- **Trace collection**: Via OpenTelemetry and Jaeger protocols
- **Trace storage**: Elasticsearch backend for Jaeger
- **Trace correlation**: Automatic correlation with logs via trace_id

### Log Aggregation

- **Log collection**: Fluent Bit collects from container logs
- **Log enrichment**: Kubernetes metadata and OpenTelemetry correlation
- **Log storage**: Loki with S3 backend for scalability
- **Log retention**: 30-day retention with automatic cleanup

## Customization

### Environment-Specific Configurations

The repository uses Kustomize for environment-specific configurations with the hub-and-spoke pattern:

**Hub Configuration:**
- Centralized storage backends (Loki, Jaeger, Thanos)
- Visualization layer (Grafana with pre-configured datasources)
- Load balancers for external access

**Spoke Configuration:**
- Collection agents (OpenTelemetry, Fluent Bit, Prometheus)
- Environment-specific instrumentation settings
- Remote write/forward configurations to hub

```bash
# Deploy hub and spoke components
kubectl apply -k hub/    # Central infrastructure
kubectl apply -k spoke/  # Collection agents
```

For different environments, you can customize the spoke configurations while maintaining a shared hub infrastructure.

### Scaling Considerations

For production deployments, consider:
- Increasing Loki component replicas
- Adjusting resource limits based on log volume
- Configuring appropriate storage classes
- Setting up monitoring for the observability stack itself

## Troubleshooting

### Common Issues

1. **Loki ingester issues**: Check persistent volume claims and storage class availability
2. **Fluent Bit parsing errors**: Verify log format and parser configurations
3. **S3 connectivity**: Ensure proper credentials and network access to S3
4. **High memory usage**: Adjust buffer sizes and batch configurations

### Validation

Use the provided validation script:

```bash
./validator.sh
```

### Logs and Debugging

Check component logs:

```bash
# Loki components
kubectl logs -l app.kubernetes.io/name=loki -n opentelemetry

# Fluent Bit
kubectl logs -l app.kubernetes.io/name=fluent-bit -n opentelemetry

# OpenTelemetry Collector
kubectl logs -l app.kubernetes.io/name=opentelemetry-collector
```

## Contributing

1. Fork the repository
2. Create a feature branch
3. Make your changes
4. Test with the validation script
5. Submit a pull request

## License

This project is licensed under the terms specified in the repository.

## Support

For issues and questions:
- Check the troubleshooting section
- Review component logs
- Open an issue in the repository

---

**Note**: This observability stack is designed for OpenShift/Kubernetes environments and includes enterprise-ready features like OAuth authentication, persistent storage, and scalable architectures.
