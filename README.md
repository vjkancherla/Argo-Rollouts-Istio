# Argo Rollouts with Istio

This repository demonstrates progressive delivery patterns using Argo Rollouts with Istio service mesh.

## 🚀 Quick Start

1. Install prerequisites: `./install-pre-requisites.sh`
2. Choose your deployment strategy below

## 📖 Documentation

### 🔄 Canary Deployments
- [Basic Canary Guide](docs/canary/argo_rollouts_istio_canary_guide.md) - Introduction to canary deployments
- [Canary with Analysis](docs/canary/argo_rollouts_istio_canary_analysis_guide.md) - Automated analysis and rollback
- [Canary with Prometheus](docs/canary/argo_rollouts_istio_canary_prometheus_guide.md) - Prometheus-based analysis
- [Header Routing Lifecycle](docs/canary/docs/canary/istio_header_routing_canary_lifecycle.md) - Traffic routing details

### 🔵🟢 Blue-Green Deployments
- [Blue-Green Guide](docs/blue-green/docs/blue-green/argo_rollouts_istio_blue_green_guide.md) - Zero-downtime deployments

## 🏗️ Examples

### Canary Examples
- [Basic Canary](examples/canary/basic/) - Simple canary deployment
- [With Analysis](examples/canary/with-analysis/) - Automated analysis templates
- [With Prometheus](examples/canary/with-prometheus-analysis/) - Prometheus metrics analysis

### Blue-Green Examples
- [Basic Blue-Green](examples/blue-green/basic/) - Simple blue-green deployment

## 📊 Observability

- [Prometheus Setup](observability/prometheus/) - Metrics collection
- [Grafana Setup](observability/grafana/) - Visualization dashboards

## 🔍 Logs

Example logs and troubleshooting information can be found in the [logs/](logs/) directory.