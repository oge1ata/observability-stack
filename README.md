# Observability Stack
## Platform Engineering Portfolio — Project 2

A full observability stack deployed on a local Kubernetes cluster, built as part of a 
platform engineering portfolio. Covers the three pillars of observability — metrics, 
logs, and alerting — using the same tools found in production engineering environments.

---

## What this project covers

Most engineers know when something is broken because a user tells them.
This project is about knowing before the user does.

The stack gives you three things:
- **Metrics** — numbers over time. Request rate, latency, error rate, resource usage.
- **Logs** — what actually happened and when, across every pod in the cluster.
- **Alerts** — automated notifications when something crosses a threshold.

Without these, operating a production system is guesswork.

---

## Stack

| Tool | Purpose |
|---|---|
| Prometheus | Scrapes and stores metrics from the cluster and application |
| Grafana | Dashboards and alerting — the main interface for seeing what's happening |
| Loki | Log aggregation — like Prometheus but for logs |
| Promtail | Agent that ships logs from pods into Loki |
| kube-prometheus-stack | Helm chart bundling Prometheus, Grafana, and Alertmanager |

---

## Cluster

Built on top of the cluster from [Project 1](https://github.com/oge1ata/demo-app).

| Node | Role |
|---|---|
| k8s-control | Control plane |
| k8s-worker1 | Worker |
| k8s-worker2 | Worker |

3-node kubeadm cluster running on Multipass VMs (Apple Silicon / arm64).

---

## What's in this repo