# Jenkins setup and troubleshooting

How the Jenkins server is built, how to reach it, how the job is configured, and the failures that actually happened while building this capstone. Pipeline stages are described in [pipeline.md](pipeline.md); trigger behaviour in [jenkins-triggers.md](jenkins-triggers.md).

## The server

| Item | Value |
|---|---|
| Host | EC2 `t3.medium`, Ubuntu 22.04, 20 GB disk, default VPC in `ap-south-1` |
| Identity | Instance profile `jenkins-capstone-profile` (role `jenkins-capstone-role`), see [architecture.md](architecture.md#iam-boundaries) |
| Jenkins | LTS from the official apt repository, Java 21 |
| Access | SSM Session Manager (no SSH needed) |

The host is created by hand and is not part of `terraform destroy`, so builds, credentials and the job survive stopping and starting the instance.

## Build the host from scratch

1. **Launch the instance** with the instance profile attached, a security group that allows ports 22 and 8080 from your own IP address (never `0.0.0.0/0`), and a 20 GB root volume. The role needs `AmazonSSMManagedInstanceCore` so Session Manager works.
2. **Install packages:**
   ```bash
   sudo apt update
   sudo apt install -y fontconfig openjdk-21-jre docker.io python3-venv python3-pip unzip
   ```
   Jenkins 2.5xx requires Java 21; Java 17 makes the service exit immediately.
3. **Add the Jenkins apt repository** using the 2026 signing key. The older `jenkins.io-2023.key` has expired. On Ubuntu 22.04 (apt 2.4) the `signed-by` form failed even though the key was valid, so put the key in the trusted keys directory instead:
   ```bash
   curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io-2026.key | sudo gpg --dearmor -o /etc/apt/trusted.gpg.d/jenkins.gpg
   echo "deb https://pkg.jenkins.io/debian-stable binary/" | sudo tee /etc/apt/sources.list.d/jenkins.list
   sudo apt update && sudo apt install -y jenkins
   ```
4. **Let Jenkins use Docker and start the services:**
   ```bash
   sudo usermod -aG docker jenkins
   sudo systemctl enable --now docker jenkins
   sudo systemctl restart jenkins
   ```
5. **Give the `jenkins` user passwordless sudo.** The Ansible stage needs root to configure the host:
   ```bash
   echo "jenkins ALL=(ALL) NOPASSWD: ALL" | sudo tee /etc/sudoers.d/jenkins-ansible
   sudo chmod 440 /etc/sudoers.d/jenkins-ansible
   sudo visudo -c
   ```
6. **Install Terraform** (HashiCorp apt repository or the zip from releases.hashicorp.com). Do not leave a HashiCorp apt source with an expired key on the host; it breaks Ansible's `apt` module (see troubleshooting).
7. **Install AWS CLI v2 before the first build.** The first stages (account lookup, ECR login) call `aws` before Ansible has run:
   ```bash
   curl -fsSL https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip -o /tmp/awscliv2.zip
   unzip -q /tmp/awscliv2.zip -d /tmp && sudo /tmp/aws/install
   ```
   kubectl and jq are installed by the Ansible playbook on the first run, and the pipeline installs Ansible itself if it is missing. Image scanning uses the `aquasec/trivy` container, so nothing extra is installed for it.
8. **Unlock Jenkins** (initial admin password in `/var/lib/jenkins/secrets/initialAdminPassword`) and install the suggested plugins.

## Reaching the Jenkins UI

Direct SSH and port 8080 were blocked on the corporate network used for this project, so everything goes through Session Manager:

```bash
# shell on the host
aws ssm start-session --target <instance-id> --region ap-south-1

# Jenkins UI at http://localhost:8080
aws ssm start-session --target <instance-id> --region ap-south-1 \
  --document-name AWS-StartPortForwardingSession \
  --parameters '{"portNumber":["8080"],"localPortNumber":["8080"]}'
```

Leave the port-forward terminal open while you use the UI.

## Plugins

Installed: the suggested set plus **Docker Pipeline**, **Amazon ECR**, **Pipeline: AWS Steps** and **Git**.

The Kubernetes plugin is not required. `kubectl` and the AWS CLI run from shell steps and authenticate with the instance profile and the EKS access entry. Install the plugin only if you want agents that run as pods.

## Credentials

| ID | Kind | Used for |
|---|---|---|
| `slack-webhook-url` | Secret text | Failure message posted by the pipeline's `post { failure }` block. If it is missing the step is skipped with a log line. |

AWS access uses the instance profile, so there are no AWS keys in Jenkins. Never print or commit the webhook URL; regenerate it in the Slack app if it leaks.

## Job configuration

| Setting | Value |
|---|---|
| Job | `capstone-build`, Pipeline |
| Definition | Pipeline script from SCM, Git |
| Repository | `https://github.com/shinmaheshwari/devops-ci-cd-pipeline-project.git` (public, no credentials) |
| Branch specifier | `*/main` |
| Script path | `jenkins/Jenkinsfile` |
| Trigger | `pollSCM` declared in the Jenkinsfile; it registers after the first build of that file |

## Day-to-day use

1. Start the Jenkins instance, wait for it to pass its status checks, open the SSM port-forward and log in.
2. **Build Now**, or push a commit and wait for the poll. The first build creates the cluster (about 15 minutes). Jenkins is the cluster creator, so EKS gives its role admin automatically.
3. To use `kubectl` from your laptop, give your IAM principal an access entry once (commands in step 3 of the [README](../README.md#3-infrastructure)), then run `aws eks update-kubeconfig --name devops-capstone --region ap-south-1`. If you create the cluster from a laptop instead, apply with `-var jenkins_role_arn=<Jenkins role ARN>` so Jenkins gets access too.
4. When finished: disable the job, delete the app's load balancer Service, `terraform destroy`, and stop the instance. Full order and checks are in [cost-optimization.md](cost-optimization.md).

Rollback of a bad deploy: `kubectl rollout undo deployment/capstone-app -n capstone-app`.

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| Build fails immediately with `Failed to exec spawn helper` when running `git` | Jenkins has been running since before a JDK package update, so its JVM cannot start child processes | `sudo systemctl restart jenkins` |
| Service will not start; journal says `Running with Java 17 ... minimum required version (Java 21)` | Wrong Java version | `sudo apt install -y openjdk-21-jre`, then `sudo systemctl reset-failed jenkins && sudo systemctl restart jenkins`. Read the real error with `sudo journalctl -u jenkins -n 100`; `/var/log/jenkins/jenkins.log` may not exist. |
| `apt update` says the Jenkins repo is not signed (`NO_PUBKEY 7198F4B714ABFC68`) | Jenkins rotated its signing key in 2026 | Use `jenkins.io-2026.key`; if `signed-by` still fails, put the dearmored key in `/etc/apt/trusted.gpg.d/` (step 3) |
| Ansible stage fails with `Failed to update apt cache: unknown reason` | A leftover apt source with an expired key (the HashiCorp repo). Plain `apt-get` only warns; Ansible's `apt` module fails. | `sudo rm -f /etc/apt/sources.list.d/hashicorp.list` |
| Ansible stage fails with `sudo: a password is required` | `jenkins` user has no passwordless sudo | Step 5 |
| Ansible task fails with `set: Illegal option -o pipefail` | The `shell` module defaults to `/bin/sh`, not bash | `executable: /bin/bash` on the task (already in the repo) |
| Build says SUCCESS but runs old stages or an old message | Branch specifier still points at an old branch | Check the commit hash in the image tag against `git log`; set the specifier to `*/main` |
| Smoke test fails with `curl` exit code 6 | A brand-new load balancer's DNS name has not propagated yet | The smoke test retries for about 5 minutes; if it still fails, check the Service with `kubectl get svc -n capstone-app` |
| Red `git ls-remote` warning on the job's configure page | Transient check from the page, not the real checkout | Verify on the host: `sudo -u jenkins git ls-remote <repo-url> refs/heads/main` |
| UI hangs although the tunnel prints "Connection accepted" | Stale SSM port-forward session | Stop the old `session-manager-plugin` process (`lsof -i :8080`) and start a fresh tunnel |
| `kubectl` says `You must be logged in to the server` or `Forbidden` after recreating the cluster | Access entries belong to a cluster, and new ones take 30 to 60 seconds to apply | Wait and retry. The creator (Jenkins by default) gets admin through `bootstrap_cluster_creator_admin_permissions = true`. Anyone else needs an access entry, and `jenkins_role_arn` must be passed when someone other than Jenkins creates the cluster. |
| `403` / `AccessDenied` right after attaching an IAM policy | IAM changes take 30 to 60 seconds to reach an instance's credentials | Retry after a minute |
| `terraform destroy` fails with `DependencyViolation` on subnets or the VPC | The app's load balancer (created by Kubernetes, not Terraform) and its `k8s-elb-*` security group still exist | Delete the `LoadBalancer` Services first, wait until the ELB is gone, then destroy |
| A cluster appeared again after you destroyed it | `pollSCM` started a build whose Terraform stage recreated it | Disable the job while the cluster is down |
| `terraform apply` fails with `409 ResourceInUseException` on an access entry | The cluster creator already has an automatic access entry, and Terraform tried to create a second one for the same principal | Leave `jenkins_role_arn` empty when Jenkins creates the cluster ([sprint-7.md](sprint-7.md)) |
| `ResourceInUseException: Cluster already exists` | An aborted apply left a cluster that Terraform never recorded in state | `terraform import module.eks.aws_eks_cluster.main devops-capstone`, then plan. Never abort a build during Terraform Apply. |
| `Scan image` fails with CRITICAL vulnerabilities | A fixable CVE in the base image, usually from a stale cached layer | The build already uses `docker build --pull` and `apt-get upgrade`; rebuild ([sprint-7.md](sprint-7.md)) |
| `Scan image` fails with `x509: certificate signed by unknown authority` | The network does TLS inspection and blocks the scanner's database download | Run it where the pipeline runs (the Jenkins EC2 host), not behind the inspecting proxy. The stage fails closed, so nothing unscanned is pushed. |
| First stage fails with `aws: command not found` on a new host | AWS CLI v2 must exist before the first build because the early stages call it before Ansible runs | Step 7 of the host setup |

## Security notes

- The `jenkins` user has `NOPASSWD: ALL`; pipeline code effectively has root on this host.
- The instance role has broad permissions (see [architecture.md](architecture.md#iam-boundaries)).
- Jenkins is reached through Session Manager; its security group is never opened to the whole internet.
