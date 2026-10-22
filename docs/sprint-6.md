# Sprint 6 — Testing, documentation, production readiness

## Status: complete — verified with a full Jenkins run (build #23, all stages green)

## Deliverables added

| Artifact | Purpose |
|----------|---------|
| [e2e-testing.md](e2e-testing.md) | Manual + automated test checklist |
| [demo-script.md](demo-script.md) | 5–10 min viva walkthrough |
| [cost-optimization.md](cost-optimization.md) | Cost choices (10% evaluation) |
| [jenkins-triggers.md](jenkins-triggers.md) | SCM poll / webhook guidance |
| Jenkins **Pipeline validation** stage | Post-deploy cluster + metrics checks |
| Jenkins `triggers { pollSCM }` | Automated build on repo changes |

## Capstone pipeline stage mapping

| Capstone stage (task.md) | Jenkins stages |
|--------------------------|----------------|
| Build | Test, Build image, Push to ECR |
| Infrastructure | Terraform Init, Plan, Apply |
| Configuration | Ansible Configure |
| Deployment | Deploy, Post-deploy smoke test |
| Testing & monitoring | Pipeline validation, Monitoring |

## Definition of done

- [x] Automated tests in pipeline (pytest + smoke + validation stage)
- [x] E2E test documentation
- [x] Documentation index updated in README
- [x] SCM trigger documented + pollSCM in Jenkinsfile
- [x] Cost optimization document
- [x] Demo script
- [x] Final full pipeline run captured (build #23, `screenshots/sprint6-final-pipeline.png`)

## Known limitations

- Jenkins EC2 not managed by Terraform (Sprint 1 bootstrap) — see [terraform.md](terraform.md#out-of-scope-in-terraform-documented-for-taskmd).
- Grafana admin password is hard-coded for demo (`monitoring/grafana.yaml`); Grafana is `ClusterIP`, reachable only through `kubectl port-forward`.
- Slack alerts need the Jenkins credential `slack-webhook-url` (Secret text); without it the step is skipped (documented in [monitoring.md](monitoring.md)).
- `pollSCM` rebuilds on every new commit and the pipeline runs `terraform apply`, so **disable the Jenkins job while the cluster is destroyed** or a poll will recreate it.
- Submission PNGs are in `docs/screenshots/` (expected files listed in [screenshots/README.md](screenshots/README.md)).
