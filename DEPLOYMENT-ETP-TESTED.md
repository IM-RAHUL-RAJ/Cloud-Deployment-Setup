# Tested ETP Deployment Runbook

This is the filled-in runbook for the deployment that was actually completed for
this repository. It is intentionally separate from the generic
[`DEPLOYMENT.md`](DEPLOYMENT.md).

The only values not written here are passwords, JWT secrets, API keys, and SMTP
credentials. Copy those into the ignored `k8s/base/secrets.env` from a secure
source before applying Kubernetes resources.

## Tested Values

```text
AWS_REGION=ap-south-1
CLUSTER_NAME=etp-cluster-2
NODEGROUP_NAME=workers
NODE_TYPE=t3.medium
DESIRED_NODES=2
MIN_NODES=2
MAX_NODES=3
DB_HOST=database-1.clcw0sygakz6.ap-south-1.rds.amazonaws.com
DB_PORT=5432
DB_NAME=postgres
DB_USER=postgres
AUTH_PORT=3000
ORDER_PORT=8081
EXECUTOR_PORT=8082
FRONTEND_PORT=80
```

The ECR repositories used were:

```text
etp-auth-service
etp-order-service
etp-executor-service
etp-frontend
```

## 1. Set AWS Context

Run from the repository root:

```bash
cd /home/ubuntu/Documents/CD/chennai-capstone-SE5-team2

export AWS_REGION=ap-south-1
export CLUSTER_NAME=etp-cluster-2
export NODEGROUP_NAME=workers
export DB_HOST=database-1.clcw0sygakz6.ap-south-1.rds.amazonaws.com
export DB_PORT=5432
export DB_NAME=postgres
export DB_USER=postgres
export AUTH_PORT=3000
export ORDER_PORT=8081
export EXECUTOR_PORT=8082
export FRONTEND_PORT=80
export AUTH_IMAGE=etp-auth-service
export ORDER_IMAGE=etp-order-service
export EXECUTOR_IMAGE=etp-executor-service
export FRONTEND_IMAGE=etp-frontend

aws configure set region "$AWS_REGION"
aws sts get-caller-identity
```

## 2. Create the Cluster

The tested values are already stored in
[`cluster.example.yaml`](k8s/cluster.example.yaml). Run the file directly:

```bash
eksctl create cluster -f k8s/cluster.example.yaml
```

Configure `kubectl`:

```bash
aws eks update-kubeconfig \
  --region "$AWS_REGION" \
  --name "$CLUSTER_NAME"

kubectl config current-context
kubectl get nodes -o wide
```

Expected result: two `Ready` worker nodes in `etp-cluster-2`.

## 3. Install EBS CSI Storage

Kafka uses encrypted EBS gp3 storage. Install the EKS-managed addon:

```bash
eksctl create addon \
  --cluster "$CLUSTER_NAME" \
  --region "$AWS_REGION" \
  --name aws-ebs-csi-driver \
  --version v1.66.0-eksbuild.1 \
  --force
```

Verify it:

```bash
kubectl get pods -n kube-system | grep ebs
kubectl apply -f k8s/base/storageclass.yaml
kubectl get storageclass gp3
```

If the addon reports that OIDC/IAM permissions are unavailable, discover the
worker role automatically and attach the temporary test-cluster policy:

```bash
NODEGROUP=$(aws eks list-nodegroups \
  --region "$AWS_REGION" \
  --cluster-name "$CLUSTER_NAME" \
  --query 'nodegroups[0]' --output text)

NODE_ROLE_ARN=$(aws eks describe-nodegroup \
  --region "$AWS_REGION" \
  --cluster-name "$CLUSTER_NAME" \
  --nodegroup-name "$NODEGROUP" \
  --query 'nodegroup.nodeRole' --output text)

NODE_ROLE_NAME="${NODE_ROLE_ARN##*/}"

aws iam attach-role-policy \
  --role-name "$NODE_ROLE_NAME" \
  --policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy
```

For production, use OIDC/IRSA or EKS Pod Identity with a dedicated addon role
instead of adding EBS permissions to worker nodes.

## 4. Configure RDS Values

The tested RDS configuration is:

```env
DB_HOST=database-1.clcw0sygakz6.ap-south-1.rds.amazonaws.com
DB_PORT=5432
DB_NAME=postgres
DB_USER=postgres
DB_SCHEMA_TRADE=trade
DB_SCHEMA_AUTH=auth
DB_SSL=false
PGSSLMODE=no-verify
JDBC_PARAMS=&sslmode=require
```

These values are already in [`base/config.env`](k8s/base/config.env) for this
project. For a different project, replace them with that project's RDS values.

Allow the EKS node security group to reach RDS on TCP `5432`. The following
commands discover the IDs automatically:

```bash
RDS_INSTANCE_ID=$(aws rds describe-db-instances \
  --region "$AWS_REGION" \
  --query "DBInstances[?Endpoint.Address=='$DB_HOST'].DBInstanceIdentifier | [0]" \
  --output text)

NODE_INSTANCE_ID=$(aws ec2 describe-instances \
  --region "$AWS_REGION" \
  --filters "Name=tag:eks:nodegroup-name,Values=$NODEGROUP_NAME" \
            "Name=instance-state-name,Values=running" \
  --query 'Reservations[].Instances[0].InstanceId | [0]' \
  --output text)

RDS_SECURITY_GROUP_ID=$(aws rds describe-db-instances \
  --region "$AWS_REGION" \
  --db-instance-identifier "$RDS_INSTANCE_ID" \
  --query 'DBInstances[0].VpcSecurityGroups[0].VpcSecurityGroupId' \
  --output text)

EKS_NODE_SECURITY_GROUP_ID=$(aws ec2 describe-instances \
  --region "$AWS_REGION" \
  --instance-ids "$NODE_INSTANCE_ID" \
  --query 'Reservations[0].Instances[0].SecurityGroups[0].GroupId' \
  --output text)

aws ec2 authorize-security-group-ingress \
  --group-id "$RDS_SECURITY_GROUP_ID" \
  --protocol tcp \
  --port 5432 \
  --source-group "$EKS_NODE_SECURITY_GROUP_ID" || \
  echo 'The rule may already exist.'
```

Test RDS connectivity from Kubernetes without putting the password in shell
history:

```bash
kubectl run rds-test --rm -it --restart=Never \
  --image=postgres:16 \
  --env="DB_HOST=$DB_HOST" \
  --env="DB_PORT=$DB_PORT" \
  --env="DB_NAME=$DB_NAME" \
  --env="DB_USER=$DB_USER" \
  -- bash
```

Inside the temporary pod, the local non-secret connection variables are now
available. The password is still requested interactively and is not passed as a
Kubernetes argument:

```bash
read -rsp 'RDS password: ' PGPASSWORD; echo
PGPASSWORD="$PGPASSWORD" pg_isready \
  -h "$DB_HOST" -p "$DB_PORT" -U "$DB_USER" -d "$DB_NAME"
unset PGPASSWORD
```

## 5. Create Kubernetes Secrets

Do not commit the populated file:

```bash
cp k8s/base/secrets.env.example k8s/base/secrets.env
${EDITOR:-vi} k8s/base/secrets.env
```

Fill these keys from the project's secure secret source:

```env
DB_PASSWORD=
JWT_SECRET=
FAUXNANCE_API_KEY=
GEMINI_API_KEY=
SMTP_USER=
SMTP_PASS=
```

## 6. Create and Push ECR Images (Skip if Already Pushed)

Skip this entire section if the four ECR repositories already exist and the
current images have already been pushed. Before continuing, verify that
[`base/kustomization.yaml`](k8s/base/kustomization.yaml) references those existing
repositories and the intended image tags.

The AWS account ID is discovered automatically:

```bash
AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
REGISTRY="$AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com"

for repo in "$AUTH_IMAGE" "$ORDER_IMAGE" "$EXECUTOR_IMAGE" "$FRONTEND_IMAGE"; do
  aws ecr describe-repositories \
    --region "$AWS_REGION" \
    --repository-names "$repo" >/dev/null 2>&1 || \
  aws ecr create-repository \
    --region "$AWS_REGION" \
    --repository-name "$repo" \
    --image-scanning-configuration scanOnPush=true \
    --encryption-configuration encryptionType=AES256
done

aws ecr get-login-password --region "$AWS_REGION" \
  | docker login --username AWS --password-stdin "$REGISTRY"

./run.sh -y

declare -A IMAGE_MAP=(
  ["$AUTH_IMAGE"]=auth-service
  ["$ORDER_IMAGE"]=order-service
  ["$EXECUTOR_IMAGE"]=executor-service
  ["$FRONTEND_IMAGE"]=frontend
)

for repo in "${!IMAGE_MAP[@]}"; do
  docker tag "${IMAGE_MAP[$repo]}:latest" "$REGISTRY/$repo:latest"
  docker push "$REGISTRY/$repo:latest"
done
```

Update the ECR registry values in [`base/kustomization.yaml`](k8s/base/kustomization.yaml)
if the AWS account or image tag differs.



Validate and apply:

```bash
kubectl kustomize k8s/overlays/four-lb >/tmp/etp-rendered.yaml
kubectl apply --dry-run=server -k k8s/overlays/four-lb
kubectl apply -k k8s/overlays/four-lb
```

Watch the rollout:

```bash
kubectl get pods -w
kubectl get svc
kubectl rollout status deployment/auth-service --timeout=180s
kubectl rollout status deployment/order-service --timeout=180s
kubectl rollout status deployment/executor-service --timeout=180s
kubectl rollout status deployment/frontend --timeout=180s
```

## 8. Important Cleanup

The local Docker Compose stack and EKS stack must not run simultaneously against
a small RDS instance, or PostgreSQL connection slots can be exhausted:

```bash
docker compose down
```

The four-LoadBalancer setup is for the initial test. The ingress overlay is the
preferred browser-facing deployment because all services share one origin.
