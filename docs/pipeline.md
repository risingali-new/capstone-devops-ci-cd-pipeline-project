# Pipeline

## Status: complete — verified end-to-end, live app reachable via LoadBalancer

## Stage diagram

```
Checkout -> Test -> Build image -> Scan image -> Push to ECR -> Terraform (Init/Plan/Apply) -> Ansible Configure -> Deploy -> Post-deploy smoke test -> Monitoring -> Pipeline validation
```

CI stages come first (fail fast): a failing test or a CRITICAL vulnerability stops the run before Terraform. `Terraform Apply` runs only when `terraform plan -detailed-exitcode` reports changes. Why and how: [sprint-7.md](sprint-7.md).

## What Deploy does

1. `aws eks update-kubeconfig` — points kubectl at the current cluster
2. Applies `metrics-server` (pinned to `v0.7.2`, idempotent) — **not included by default on EKS**,
   required for HPA to report real CPU numbers instead of `<unknown>`
3. Applies `kubernetes/*.yaml` (namespace, deployment, service, hpa)
4. `kubectl set image` — swaps in the image tag just built/pushed this run,
   rather than whatever's checked into `deployment.yaml`
5. `kubectl rollout status` — waits for the rollout to actually finish
6. Smoke test stage polls for the LoadBalancer hostname (up to 5 min) then
   curls `/health` through the real public endpoint

## What Monitoring and Pipeline validation do

**Monitoring** applies the manifests in `monitoring/` (namespace, kube-state-metrics, node-exporter,
Prometheus RBAC, config and deployment, Grafana dashboards, datasource and deployment) and waits up to
120 seconds each for the kube-state-metrics, Prometheus and Grafana rollouts. Details: [monitoring.md](monitoring.md).

**Pipeline validation** is the last check. It confirms the `capstone-app` deployment is available,
lists the monitoring pods, then reads `/health` and `/metrics` through the public load balancer and
checks that `app_requests_total` is present.

## Rollback

```bash
kubectl rollout undo deployment/capstone-app -n capstone-app
kubectl rollout status deployment/capstone-app -n capstone-app
```

## Verifying it worked

```bash
kubectl get pods -n capstone-app        # both replicas Running, 1/1
kubectl get hpa -n capstone-app         # cpu: X%/70%, not <unknown>
kubectl get svc capstone-app -n capstone-app   # LoadBalancer hostname assigned
curl http://<hostname>/health
```

## Lessons learned

1. **Metrics Server is not part of default EKS.** HPA will show `<unknown>`
   for CPU targets indefinitely without it — no error, just silently useless
   autoscaling. Installing it as an idempotent step inside the Deploy stage
   (rather than a one-time manual setup step someone has to remember) means
   it's automatically present on every fresh cluster, including after a
   destroy/recreate cycle.

2. **A stale Jenkins job Branch Specifier caused several "successful" but
   wrong builds.** Back in Sprint 3, the job was pointed at `*/sprint3-review`
   to test that PR branch before merging — and never switched back to
   `*/main` afterward. Every subsequent build kept quietly rebuilding the old
   branch, showing `SUCCESS` with the *old* pipeline stages and post-message
   (`"Infra applied and pushed..."` instead of the new `"Deployed ... to
   EKS"`), which looked like a real result but was stale. The tell was
   comparing the commit hash in the image tag against the latest commit on
   `main` — worth checking that match any time a build's behavior doesn't
   match what was just pushed.

3. **EKS access entries don't survive `terraform destroy` / recreate**, even
   for the Jenkins role (which IS managed by Terraform and gets its access
   entry recreated automatically) — but the *cluster creator's* own access
   depends on `bootstrap_cluster_creator_admin_permissions`, which wasn't set.
   This caused the exact same "must be logged in to the server" error twice,
   on two separate destroy/recreate cycles, before being fixed by adding
   `bootstrap_cluster_creator_admin_permissions = true` to the cluster's
   `access_config` block — worth remembering that a fix flagged as a
   "follow-up, not yet addressed" in one sprint's docs is a real, recurring
   cost if left unaddressed into the next.

## Measured run time

Build #31 (`screenshots/sprint7-pipeline-stage-order.png`) with the cluster already running:

| Stage | Time |
|---|---|
| Checkout | 2 s |
| Test (pytest) | 3 s |
| Build image | 5 s |
| Scan image (Trivy) | 1 s |
| Push to ECR | 1 s |
| Terraform Init | 25 s |
| Terraform Plan | 7 s |
| Terraform Apply | 6 s |
| Ansible Configure | 26 s |
| Deploy | 32 s |
| Post-deploy smoke test | 1 min 17 s |
| Monitoring | 19 s |
| Pipeline validation | 4 s |
| **Total** | **about 3 min 30 s** |

Notes:

- The four CI stages take about 10 s in total, so a bad commit is rejected within seconds and never reaches Terraform.
- Terraform Apply ran in this build. When the plan has no changes the stage is skipped (build #34, `screenshots/sprint7-terraform-apply-skipped.png`).
- The smoke test is the longest stage because it waits for the load balancer's hostname, sleeps 30 s, then retries `/health`.
- Creating the cluster from nothing is separate and adds roughly 10 to 15 minutes (`terraform apply`).

## Related docs

- [sprint-7.md](sprint-7.md): why the pipeline is ordered this way, with evidence.
- [monitoring.md](monitoring.md): Prometheus, Grafana, alert rules and the Slack failure message.
- [jenkins-triggers.md](jenkins-triggers.md): how builds start, including `pollSCM`.
- [jenkins.md](jenkins.md): Jenkins host setup and troubleshooting.
- [cost-optimization.md](cost-optimization.md): what costs money and the safe teardown order.
