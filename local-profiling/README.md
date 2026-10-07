# Local profiling test

eBPF profiler -> OTel collector -> Pyroscope, via Docker Compose.

## Run (Linux host or Linux VM)

```bash
docker compose up
docker compose logs -f collector profiler
```

Open http://localhost:4040 and pick a `process_cpu` profile type.

## macOS / Docker Desktop

The eBPF profiler does not run there. You can still check that the collector
starts with the profiles pipeline and connects to Pyroscope:

```bash
docker compose up pyroscope collector
docker compose logs collector
```

## Validate the real manifests

Extract `spec.config` from `spoke/agent.yml` / `hub/gateway.yml` into a file, then:

```bash
docker run --rm -e OTEL_AUTH_TOKEN=x -e K8S_NODE_NAME=n -v "$PWD":/c \
  otel/opentelemetry-collector-contrib:0.149.0 \
  validate --feature-gates=service.profilesSupport --config=/c/<file>.yaml
```

Without the feature gate the profiles pipeline is rejected.

## Files

| File | Role |
|---|---|
| `profiler.yaml` | eBPF profiler config, sends OTLP to the collector |
| `collector.yaml` | Collector with a `profiles` pipeline to Pyroscope |
| `pyroscope.yaml` | Pyroscope config (maps process name to `service_name`) |
