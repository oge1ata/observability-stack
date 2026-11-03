#What I've built so far

##📊 kube-prometheus-stack
Installed kube-prometheus-stack with Helm into a `monitoring` namespace. One chart, about six components:
Prometheus — scrapes metrics over HTTP every 15–30s and stores them as time series
Alertmanager — receives firing alerts and routes them (Slack, email, PagerDuty)
Grafana — the UI for dashboards and alerts
node-exporter — runs on every node, exposes CPU / memory / disk / network
kube-state-metrics — exposes Kubernetes object state ("deployment wants 3 replicas, has 2")
Prometheus Operator — lets me configure Prometheus with Kubernetes YAML (`ServiceMonitor`, `PrometheusRule`) instead of editing Prometheus config files. Same reconcile-loop idea as ArgoCD.
Had cluster and node dashboards straight away with zero app instrumentation — that's infrastructure metrics vs application metrics.

##🔌 Getting kubectl + Helm working from the Mac
`helm` wasn't installed — `brew install helm`. Installed globally, so it doesn't matter which folder I was in.
kubectl on the Mac had no cluster configured (`current-context is not set`). Copied kubeadm's admin kubeconfig off the control plane:
`multipass exec k8s-control -- sudo cat /etc/kubernetes/admin.conf > ~/.kube/config`
Running tools from the Mac (not inside the VM) matters for port-forward — the tunnel opens on the machine running the command, so it has to be the machine with the browser.
Never commit that kubeconfig — it's full cluster-admin.

##📜 Loki + Promtail
Installed `grafana/loki-stack` with Promtail enabled.
Promtail is a DaemonSet (one per node) that tails container log files and ships them to Loki with Kubernetes labels attached.
Loki only indexes labels, not the full log text — much cheaper than Elasticsearch, slower for full-text search.
Logs survive the pod — `kubectl logs` doesn't once a pod is replaced.
Note: loki-stack is deprecated and Promtail is end-of-life; Grafana Alloy is the replacement. Fine for learning, but I'd use Alloy in production.

##🐛 Debugging: Grafana in CrashLoopBackOff
Went to reconnect to Grafana and it had restarted 308 times in 45 hours.
`kubectl logs` showed the actual error on the last line:
`Datasource provisioning error: ... Only one datasource per organization can be marked as default`
Both kube-prometheus-stack (Prometheus) and loki-stack (Loki) register themselves as Grafana's default datasource. Grafana refuses to start with two.
Fixed by setting `loki.isDefault: false` in `helm/values/loki.yaml` and running `helm upgrade -f`, then restarting Grafana.
Gotcha: `helm upgrade -f` replaces earlier `--set` values, it doesn't merge — had to put `promtail.enabled: true` in the file too, or promtail would have been switched off.
Lesson: values belong in files in the repo, so the repo is the record of what's actually deployed.
Lesson: "can't connect" is often "the pod is dead". `kubectl get pods` → STATUS / RESTARTS → `kubectl logs` comes first.

##🧪 Instrumenting demo-app
Added `prometheus-fastapi-instrumentator` to `app.py` and `requirements.txt` in the Project 1 repo:
`.instrument(app)` — middleware that times and counts every request by handler, method, status
`.expose(app)` — adds a `/metrics` route that serves those numbers as plain text
(First tried pasting the Python into the terminal — it's code for the app, not shell commands.)
Pushed → GitHub Actions built the image → ArgoCD rolled it out. Project 1 doing its job.

##🐛 Debugging: port-forward dying during a rollout
`curl localhost:8080/metrics` gave `Empty reply from server`, then `network namespace for sandbox ... is closed`.
`port-forward svc/...` doesn't really go through the Service — it picks one pod and tunnels to it. I'd opened it mid-rollout, and that pod got terminated.
Also learned port-forward blocks the terminal — the curl on the next line never ran until the tunnel died. Tunnels go in their own tab.
Lesson: `kubectl get pods` (look at AGE) and `kubectl get rs` tell you if a rollout is still happening. Old ReplicaSets at 0 replicas are the rollout history `kubectl rollout undo` uses.

##🎯 Getting Prometheus to scrape the app
The CLAUDE.md said Prometheus "scrapes automatically" — it doesn't with kube-prometheus-stack. Needed a ServiceMonitor.
Changes in the Project 1 repo:
Added `labels: app: demo-app` to the Service and named the port `http`
Wrote `k8s/servicemonitor.yaml` with `release: monitoring` label (Prometheus ignores ServiceMonitors without it)
The chain: Prometheus → ServiceMonitor (`release: monitoring`) → Service (`app: demo-app`) → pods → port `http` at `/metrics`. All wired together by labels.

##🐛 Debugging: ServiceMonitor not appearing
Wrote the file, nothing happened. It was never committed — untracked on my laptop. In GitOps the file on your laptop doesn't count; Git is the source of truth.
After pushing, still `No resources found`. Compared commit hashes: GitHub had my commit, ArgoCD had last synced the one before.
ArgoCD polls GitHub (~3 min) — it can't get a webhook because the cluster is on my laptop. Forced it with the refresh annotation.
Confirmed with `up{job="demo-app"}` = 1 for all 3 pods.
Lesson: compare desired state (Git) against actual state (cluster) — that's how you debug GitOps.

##📈 Dashboard
Built a `demo-app` dashboard, exported to `dashboards/demo-app.json`:
Request rate — `sum by (handler) (rate(http_requests_total{job="demo-app", handler!="/metrics"}[5m]))`
Error rate — 4xx|5xx rate ÷ total rate (a ratio, not a count)
Latency p50 / p95 / p99 — `histogram_quantile` over `http_request_duration_highr_seconds_bucket`
CPU — `container_cpu_usage_seconds_total` per pod
Memory — `container_memory_working_set_bytes` per pod
Request rate + errors + duration = the RED method. CPU + memory = saturation. Together: Google's four Golden Signals.
Generated traffic with a curl loop hitting `/`, `/health` and `/does-not-exist` (for 404s) — empty graphs teach nothing.
Why percentiles not averages: 99 requests at 10ms and 1 at 5s averages to ~60ms and hides the one user in 100 who's suffering.
Why `rate()`: counters only ever go up. `rate()` turns "total since boot" into "per second right now".

##🐛 Debugging: latency panel showing 0–100%
Latency panel had a y-axis of 0–100% with p99 pinned flat at the top.
Real values from Prometheus were fine: p50 5ms, p95 9.6ms, p99 10ms.
Cause: Stack series was set to 100% — draws each line as a share of the total, so the top series always hits 100%. Percentiles must never be stacked (p99 already includes p50).
Fixed: Stack series Off, unit seconds, fill opacity 0.

##🔍 Things the data showed me
`/metrics` is the slowest endpoint in the app (~17.7ms avg vs ~1ms for `/`). Prometheus's own scrapes push p95/p99 up — the monitoring affects the thing being monitored. The `highr` histogram has no handler label, so they can't be filtered out of the latency panel.
p99 sat at exactly 10ms because the smallest histogram bucket is ≤10ms and almost everything lands in it. `histogram_quantile` can only interpolate inside a bucket — percentiles are only as precise as your bucket boundaries.
CPU stepped up from ~0.002 to ~0.015 cores exactly when the curl loop started — traffic and saturation moving together.
Memory steady at ~35 MiB against a 128Mi limit — headroom, no leak.

##🚨 Alerts
Wrote `alerts/rules.yaml` as a `PrometheusRule` (with `release: monitoring` label), applied with `kubectl apply`:
`DemoAppHighErrorRate` — error ratio > 5% for 2m, severity warning
`DemoAppPodsUnavailable` — available replicas < desired replicas for 1m, severity critical
`for: 2m` stops a single blip from paging someone — first defence against alert fatigue.
Didn't use `up == 0` for "pod down": a deleted pod's target disappears rather than reporting 0, so that alert would never fire. Desired vs available replicas (from kube-state-metrics) catches both crashed and missing pods.
`DemoAppHighErrorRate` has fired — the curl loop runs ~33% 404s, well over 5%.
Alerts went inactive once the curl loop stopped — no traffic means no errors.

##🔕 Built-in alerts firing on a fresh install
Alert rules page showed ~10 firing alerts I didn't write.
Watchdog — fires forever on purpose. A dead man's switch: if it ever stops arriving, the alerting pipeline itself is broken.
The rest (etcdMembersDown, TargetDown, KubeScheduler / ControllerManager / Proxy Unreachable) — 6 targets down:
kube-proxy on all 3 nodes (:10249), kube-controller-manager (:10257), kube-scheduler (:10259), etcd (:2381)
Cause: kubeadm binds those components' metrics to 127.0.0.1. Prometheus runs in a pod, so it can't reach the node's localhost. The components are healthy — just invisible.
Lesson: 10 alerts firing and none of them real is alert fatigue in miniature. Fixing or silencing noisy alerts is part of setting up alerting.

---

#Still to do

##⏳ Fix the kubeadm metrics targets
Change `--bind-address=127.0.0.1` → `0.0.0.0` in the controller-manager and scheduler static pod manifests, `--listen-metrics-urls` → `http://0.0.0.0:2381` for etcd, and `metricsBindAddress: "0.0.0.0:10249"` in the kube-proxy ConfigMap.
Trade-off: exposes metrics on the node network. Fine in a lab; in production restrict with firewall / network policy and set it in the kubeadm config so upgrades don't revert it.

##⏳ Investigate the latency spike
p99 jumped to ~9s (peak in Prometheus ~6s at 5m resolution) when it's normally ~10ms. Haven't found the cause yet.
Plan: zoom the dashboard to the spike → check request rate / CPU / memory in that window → Loki `{app="demo-app"}` for the same window → `kubectl get events -A` and restart counts.
Suspects: Mac sleeping / Multipass VMs pausing, CPU throttling at the 200m limit, node memory pressure (monitoring pods have dozens of restarts).
Write up as: symptom → what I checked → what the data showed → root cause.

##⏳ Other
Test `DemoAppPodsUnavailable` — break the deployment on purpose (bad image tag), watch it fire, revert, watch it resolve.
Fill in `helm/values/prometheus.yaml` so the kube-prometheus-stack install is reproducible from the repo.
Investigate high restart counts on monitoring pods (kube-state-metrics 22, operator 17, node-exporter 20).
