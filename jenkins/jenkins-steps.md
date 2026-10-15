# Jenkins CI/CD Deployment

This repository uses two Jenkins jobs:

- `JenkinsfileCI`: checkout, build, and push four Docker images to ECR.
- `JenkinsfileCD`: manually select an existing ECR image tag, test RDS access,
  and deploy it to the existing EKS cluster.

The CD pipeline never creates a new cluster. The tested cluster name is:

```text
capstone
```

## 1. Jenkins EC2 Machine

Both jobs use `agent any` and run directly on the Jenkins EC2 machine. No
separate AWS agent is required. The Jenkins service account must be allowed to
use Docker, and the EC2 instance role must allow ECR, EKS, and the optional
RDS/EC2 security-group operations.

The Jenkins machine must have:

```bash
aws --version
docker --version
kubectl version --client
```

The CI job needs Docker access. The CD job needs AWS CLI and kubectl access.
`eksctl` is not required by CD because the cluster already exists.

Install Jenkins plugins:

- Pipeline
- Git
- Credentials Binding
- GitHub Integration

## 2. Jenkins Credentials

Create these credentials in Jenkins:

| ID | Type | Use |
| --- | --- | --- |
| `etp-k8s-secrets-env` | Secret file | The target project's populated `k8s/base/secrets.env` |

The secret file must contain the keys expected by the application:

```env
DB_PASSWORD=...
JWT_SECRET=...
FAUXNANCE_API_KEY=...
GEMINI_API_KEY=...
SMTP_USER=...
SMTP_PASS=...
```

Never commit or print this file. The CD pipeline copies it into the workspace
only during deployment and removes it in `post.always`.

## Private GitHub Repository Credentials

If the application or deployment repository is private, configure Git credentials
in the Jenkins job. Do not use a GitHub account password; GitHub Git operations
require a Personal Access Token (PAT) or SSH key.

Recommended HTTPS setup:

1. Create a GitHub fine-grained PAT with access to the required private repository.
2. Grant repository **Contents: Read-only** permission.
3. Add a Jenkins credential with:
   - Kind: **Username with password**
   - Username: GitHub username
   - Password: the PAT value
   - ID: `github-scm`
4. In the job's **Pipeline script from SCM** configuration, select the `github-scm` credential.

For an organization repository, the organization may require PAT approval or SSO authorization.
The Jenkins checkout log should then show a successful fetch instead of an authentication error.

AWS credentials are separate: the Jenkins EC2 instance role is used for AWS CLI, ECR, and EKS operations.

## 3. Create the CI Job

Create a Jenkins Pipeline job named `etp-ci`:

1. Open Jenkins at `http://localhost:8080`.
2. Choose **New Item** and select **Pipeline**.
3. Select **Pipeline script from SCM**.
4. Select **Git**.
5. Set the application repository URL.
6. Set the branch to `*/main`.
7. Set the script path to `JenkinsfileCI`.
8. Save the job.

CI stages:

1. Checkout SCM.
2. Validate Docker, AWS, and build contexts.
3. Discover the AWS account and ECR registry.
4. Create missing ECR repositories.
5. Build auth, order, executor, and frontend images.
6. Push all four images with a unique tag.

CI has no image-tag parameter. It always generates a simple version/build tag
and prints it at the end of the build log. Build 3 produces:

```text
1.3
```

Copy the printed `IMAGE_TAG=1.3` value into the CD job.

The CI console prints the image tag. Copy that tag for the CD job, for example:

```text
247-a1b2c3d4e5f6
```

## 4. GitHub Webhook for CI

The Jenkins server must be reachable by GitHub. `localhost:8080` works only from
the Jenkins host itself; GitHub cannot call a private localhost address.

Expose Jenkins through a secure public HTTPS URL, then in GitHub:

1. Open **Settings > Webhooks > Add webhook**.
2. Set the Payload URL to:

```text
https://<public-jenkins-host>/github-webhook/
```

3. Set content type to `application/json`.
4. Select **Just the push event**.
5. Enable the webhook.
6. In Jenkins job configuration, enable **GitHub hook trigger for GITScm polling**.

Pushes to the configured branch will then trigger `etp-ci`.

## 5. Create the CD Job

Create a second Jenkins Pipeline job named `etp-cd`:

1. Choose **New Item** and select **Pipeline**.
2. Select **Pipeline script from SCM**.
3. Use the deployment repository:

```text
https://github.com/IM-RAHUL-RAJ/Cloud-Deployment-Setup.git
```

   CD checks out this repository only because it needs the `k8s/` manifests,
   `generated/application.yml`, and `JenkinsfileCD`. It does not rebuild or
   push application images.
4. Set the branch to `*/main`.
5. Set the script path to `JenkinsfileCD`.
6. Save the job.

Do not enable an automatic push webhook for CD. CD is a manual promotion step.

## 6. Run CD Manually

Click **Build with Parameters** and enter the image tag printed by `etp-ci`:

```text
AWS_REGION=ap-south-1
EKS_CLUSTER_NAME=capstone
RDS_HOST=<actual RDS endpoint>
RDS_PORT=5432
RDS_DATABASE=postgres
RDS_USER=postgres
IMAGE_TAG=<tag printed by etp-ci>
K8S_OVERLAY=k8s/overlays/four-lb
K8S_NAMESPACE=default
AUTO_AUTHORIZE_RDS=true
```

The four public URL values must already be correct in:

```text
k8s/overlays/four-lb/urls.env
```

## Kubernetes Cleanup Before CD

If old application workloads are still running or consuming RDS connections,
open [`prereq.md`](prereq.md) and run its kubectl cleanup commands first.

The prerequisite uses the existing kubectl context and removes only the auth,
order, executor, frontend, and Kafka workloads. It does not configure AWS,
delete the cluster, delete PVCs, or remove Kubernetes Secrets.

After cleanup, run CD with the CI-generated `IMAGE_TAG`.

## 7. CD Execution Order

`JenkinsfileCD` runs these stages in order:

1. Checkout the SCM repository.
2. Require a selected `IMAGE_TAG` from CI.
3. Verify that `capstone` exists and is `ACTIVE`.
4. Configure kubectl for the existing cluster.
5. Verify worker nodes and the EBS CSI addon.
6. Update the workspace `config.env` with the Jenkins RDS parameters.
7. Run an RDS readiness test from a temporary PostgreSQL client pod.
8. If the test succeeds, continue immediately.
9. If it fails and `AUTO_AUTHORIZE_RDS=true`, discover the exact RDS and EKS
   security groups using both cluster and nodegroup tags, authorize TCP `5432`,
   and retry the test.
10. Verify all four selected ECR image tags exist.
11. Copy the Jenkins-managed Kubernetes Secret file into the workspace.
12. Run `kubectl apply --dry-run=server`.
13. Apply the Kustomize overlay.
14. Set all four Deployments to the selected ECR tag.
15. Wait for all rollouts to become ready.
16. Delete the temporary workspace Secret file.

The temporary RDS pod is not an application pod. It is only a short-lived
PostgreSQL client used to test the network path from EKS to RDS. It is deleted
automatically after the test.

## 8. Why the RDS Step Can Fail

The RDS lookup requires the Jenkins parameter `RDS_HOST` to exactly match the
RDS endpoint returned by AWS. A stale hostname returns no RDS instance ID.

The EKS node lookup requires these tags:

```text
aws:eks:cluster-name=capstone
eks:nodegroup-name=workers
```

The CD pipeline checks both tags so it cannot accidentally select a worker from
another cluster with the same nodegroup name.

If RDS is in another VPC, a security-group reference may not work directly. The
VPC route, peering or Transit Gateway route, NACLs, and CIDR-based RDS ingress
must be configured separately.

## 9. Files Used

```text
JenkinsfileCI                         CI pipeline
JenkinsfileCD                         Manual CD pipeline
k8s/base/config.env                   Non-secret runtime configuration
k8s/base/generated/application.yml   Shared structured configuration
k8s/base/secrets.env                  Jenkins-managed secret file, never Git
k8s/base/kustomization.yaml           ECR image substitutions
k8s/overlays/four-lb/urls.env         Public service URLs
```

The existing EKS cluster must be ready before CD:

```bash
aws eks update-kubeconfig --region ap-south-1 --name capstone
kubectl get nodes
kubectl get pods -n kube-system | grep ebs
```
