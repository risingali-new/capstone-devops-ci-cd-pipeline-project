# Monitoring — Sprint 5

## Status: implemented — verify targets, dashboard, and alerts after Jenkins run

## What gets deployed

| Component | Namespace | Purpose |
|---|---|---|
| Prometheus | `monitoring` | Scrapes app pods, kube-state-metrics, node-exporter |
| kube-state-metrics | `monitoring` | Cluster object metrics (pods ready, deployments, etc.) |
| node-exporter | `monitoring` | **Node** CPU/memory (DaemonSet on workers) |
| Grafana | `monitoring` | ClusterIP (port-forward only), datasource + **Capstone — App & Cluster** dashboard |
| Alert rules | Prometheus config | App down, high request rate, high node CPU |

The Jenkins **Monitoring** stage applies all `monitoring/*.yaml` files after the app smoke test.

## Grafana dashboard

Provisioned automatically: folder **Capstone** → **Capstone — App & Cluster**

Panels include app scrape up-count, request rate, ready pods (kube-state-metrics), and node CPU %.

Grafana is a `ClusterIP` service on purpose: the dev password is in this repo, and a public load
balancer would expose it (and block `terraform destroy`). Reach it with a port-forward:

```bash
kubectl port-forward svc/grafana -n monitoring 3000:80
# http://localhost:3000 — login admin / capstone-dev (dev-only)
```

Screenshot for submission: [docs/screenshots/README.md](screenshots/README.md) (`sprint5-grafana-dashboard.png`).

## Verify Prometheus targets

```bash
kubectl port-forward svc/prometheus -n monitoring 9090:9090
# http://localhost:9090/targets
# Expect UP: kubernetes-pods (capstone-app), kube-state-metrics, node-exporter
```

## Alert runbook

| Alert | Meaning | Test |
|-------|---------|------|
| `CapstoneAppDown` | Cannot scrape an app pod (`up == 0`) | Break scraping on one live pod: `kubectl annotate pod <pod> -n capstone-app prometheus.io/port=9999 --overwrite`, wait ~3m, check Prometheus **Alerts** (Pending, then Firing). Reset with `kubectl delete pod <pod> -n capstone-app`. |
| `CapstoneHighRequestRate` | Demo threshold exceeded | Load-test or lower threshold temporarily |
| `NodeHighCPU` | Worker CPU > 85% for 10m | Stress node or adjust threshold |

Alerts appear in Prometheus UI (**Alerts** tab). Alertmanager routing is not deployed — add Alertmanager + Slack in a future iteration.

> Scaling the deployment to 0 does **not** fire `CapstoneAppDown`: with no pods there is no `up` series at all, so
> `up == 0` never matches. That is why the test above breaks scraping on a pod that still exists.

## Jenkins failure notifications

On pipeline **failure**, Jenkins:

1. Logs guidance to check Grafana/Prometheus.
2. Posts a message to Slack with `curl`, using the Jenkins credential **`slack-webhook-url`**
   (kind **Secret text**, value = a Slack Incoming Webhook URL). The URL is never committed to the repo.

If the credential does not exist the step is skipped with a log line, so a missing credential never hides the real failure.
Alternative: **Email Extension** plugin with `emailext` in `post { failure }`.

Quick test without running the whole pipeline: create a throwaway Pipeline job with a stage that runs `exit 1`
and the same `post { failure { ... } }` block, build it once, and check the Slack channel.

## Definition of done

- [ ] `kubectl get pods -n monitoring` — prometheus, grafana, kube-state-metrics Running; node-exporter on each node
- [ ] Prometheus targets: app + kube-state-metrics + node-exporter **UP**
- [ ] Grafana dashboard **Capstone — App & Cluster** visible
- [ ] Test alert `CapstoneAppDown` once (scale to 0)
- [ ] Slack failure message received (credential `slack-webhook-url`)

## Next: Sprint 6

See [sprint-6.md](sprint-6.md) and [e2e-testing.md](e2e-testing.md).
