# Project 2 — Observability Stack
## Platform Engineering Portfolio — Project 2

---

## What this project is

Adding a full observability stack to the existing cluster from Project 1.
The goal: instrument the demo-app to expose real metrics, collect logs,
build dashboards, and set up alerts — so you can see what's actually
happening inside the cluster instead of guessing.

This is Project 2 of a 5-project platform engineering portfolio.

---

## The three pillars of observability

**Metrics** — numbers over time. Request count, latency, error rate, CPU usage.
Collected by Prometheus, visualised in Grafana.

**Logs** — what actually happened and when. Collected by Loki,
queried in Grafana.

**Alerts** — automated notifications when something crosses a threshold.
Configured in Grafana or Alertmanager.

Without these three things, the only way to know something is wrong
is when a user tells you.

---

## Stack

| Tool | What it does |
|---|---|
| Prometheus | Scrapes and stores metrics from the cluster and app |
| Grafana | Dashboards and alerting UI — this is what you look at |
| Loki | Log aggregation — like Prometheus but for logs |
| kube-prometheus-stack | Helm chart that installs Prometheus + Grafana + Alertmanager together |
| Promtail | Agent that ships logs from pods into Loki |

---

## Cluster context (inherited from Project 1)

| Thing | Detail |
|---|---|
| Cluster | 3-node kubeadm cluster on Multipass VMs (Apple Silicon) |
| Control plane IP | 192.168.252.3 |
| App | demo-app (FastAPI) running in default namespace |
| GitOps | ArgoCD watching GitHub repo, auto-sync enabled |
| CI/CD | GitHub Actions building and pushing on every push to master |

Shell into control plane:
```bash
multipass shell k8s-control
```

---

## What to build, in order

**Step 1 — Install kube-prometheus-stack via Helm**
```bash
helm repo add prometheus-community \
  https://prometheus-community.github.io/helm-charts
helm repo update
helm install monitoring prometheus-community/kube-prometheus-stack \
  -n monitoring --create-namespace
```
Verify all pods are running:
```bash
kubectl get pods -n monitoring
```

**Step 2 — Access Grafana**
```bash
kubectl port-forward svc/monitoring-grafana -n monitoring 3000:80
```
Open http://localhost:3000
Default credentials: admin / prom-operator

**Step 3 — Install Loki + Promtail**
```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm install loki grafana/loki-stack -n monitoring \
  --set promtail.enabled=true
```

**Step 4 — Instrument demo-app**
Add the prometheus-fastapi-instrumentator library to demo-app.
This exposes a /metrics endpoint that Prometheus scrapes automatically.
```python
from prometheus_fastapi_instrumentator import Instrumentator
Instrumentator().instrument(app).expose(app)
```

**Step 5 — Build a Grafana dashboard**
Create a dashboard showing:
- Request rate (requests per second)
- Latency (p50, p95, p99)
- Error rate (4xx and 5xx responses)
- Pod CPU and memory usage

**Step 6 — Set up an alert**
Create an alert rule in Grafana that fires when:
- Error rate exceeds 5% for more than 2 minutes
- Or pod is down

---

## Definition of Done

- Prometheus scraping metrics from demo-app and cluster nodes
- Grafana dashboard showing request rate, latency, and error rate
- Loki collecting logs from demo-app pods
- At least one alert rule configured
- Everything deployed via Helm, values committed to the repo

---

## Why this matters for interviews

"How would you debug a latency spike in production?"
Without observability the answer is guessing.
With this project the answer is:
- Check the Grafana dashboard for the latency spike timestamp
- Cross-reference with Loki logs for errors at that time
- Check Prometheus for resource saturation on the affected node
- Follow the data to the root cause

That's the answer interviewers at ITV, Charlotte Tilbury, and IBM want to hear.

---

## Useful debugging commands

```bash
# Check all monitoring pods are healthy
kubectl get pods -n monitoring

# Check Prometheus targets (what it's scraping)
kubectl port-forward svc/monitoring-kube-prometheus-prometheus \
  -n monitoring 9090:9090
# Then open http://localhost:9090/targets

# Check Loki is receiving logs
kubectl logs -n monitoring -l app=loki --tail=50

# Check demo-app metrics endpoint is working
kubectl exec -it <demo-app-pod> -- curl http://localhost:8000/metrics
```

---

## Next project after this

Project 3 — Terraform on GCP
Provision a GKE cluster from scratch using Terraform.
Write modules for cluster, node pools, VPC, and IAM.
Deploy the ArgoCD pipeline and observability stack on top of it.