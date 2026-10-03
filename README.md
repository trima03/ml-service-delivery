# ML Service Delivery

Deployment and observability configuration for the companion [Retail Demand Forecasting](https://github.com/trima03/demand-forecasting-ml) service. It includes a hardened local stack for repeatable demos and Kubernetes manifests for a small cluster.

## What this demonstrates

- Container deployment with non-root execution, read-only filesystem and dropped Linux capabilities.
- Liveness/readiness probes, resource requests/limits, two replicas and CPU-based autoscaling.
- Prometheus request, error-rate and latency monitoring, with actionable alert rules.
- A provisioned Grafana dashboard and Prometheus datasource.
- CI validation for Docker Compose and Kubernetes manifests.

This repository contains deployment configuration; the forecasting API and its model artifact are built in the companion repository. The local Compose stack pulls its image from GHCR.

## Start the local monitoring stack

Prerequisites: Docker Engine with the Compose plugin, and a published API image.

```bash
cp .env.example .env
# Edit .env: set FORECAST_IMAGE to your API image and choose a unique Grafana password.
docker compose up -d
```

Open the API at `http://localhost:8000/docs`, Prometheus at `http://localhost:9090`, and Grafana at `http://localhost:3000`. The dashboard is provisioned automatically. The forecast API exports Prometheus metrics at `/metrics`.

## Deploy to Kubernetes

Apply the manifests to a cluster that has a metrics server for the HPA:

```bash
kubectl apply -f k8s/
kubectl rollout status deployment/demand-forecast-api
kubectl get hpa demand-forecast-api
```

The pod uses HTTP startup, liveness and readiness probes, has no service-account token, runs as a non-root UID, and has CPU/memory requests and limits. Metrics annotations let compatible Prometheus installations discover the service. Add a network policy and ingress appropriate to your cluster before exposing it outside the cluster.

## Alerts

- `ForecastApiUnavailable`: scrape target has been down for 2 minutes.
- `ForecastApiHighErrorRate`: server errors exceed 5% for 5 minutes.
- `ForecastApiSlowRequests`: p95 latency exceeds one second for 10 minutes.

Thresholds are demo defaults and should be tuned against real service-level objectives.

## Repository layout

```text
compose.yaml
prometheus/   scrape config and alert rules
grafana/      provisioned dashboard and datasource
k8s/          Deployment, Service and HPA
.github/      deployment configuration validation
```

## Tradeoffs

The demo targets a small single cluster. It does not include cloud-specific Terraform, TLS/ingress, an alert notification receiver, or a managed secret store. Those should be added for a real production environment.
