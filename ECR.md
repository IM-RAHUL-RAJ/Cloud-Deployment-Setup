# ECR Image Deployment

This guide creates private Amazon Elastic Container Registry repositories, builds the four application images, pushes them, and updates the Kubernetes image references.

Run every command from the application repository root.

## Prerequisites

```bash
aws --version
docker --version
aws sts get-caller-identity
```

Set the AWS region once:

```bash
export AWS_REGION=ap-south-1
aws configure set region "$AWS_REGION"
```

The AWS account ID is discovered automatically. Do not copy it manually.

## Create ECR Repositories

This is safe to rerun. Existing repositories are preserved.

```bash
set -e

for repo in etp-auth-service etp-order-service etp-executor-service etp-frontend; do
  if aws ecr describe-repositories \
    --region "$AWS_REGION" \
    --repository-names "$repo" >/dev/null 2>&1; then
    echo "ECR repository already exists: $repo"
  else
    aws ecr create-repository \
      --region "$AWS_REGION" \
      --repository-name "$repo" \
      --image-scanning-configuration scanOnPush=true \
      --encryption-configuration encryptionType=AES256 \
      --query 'repository.repositoryUri' \
      --output text
  fi
done
```

## Authenticate Docker

```bash
AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
REGISTRY="$AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com"

echo "Using ECR registry: $REGISTRY"

aws ecr get-login-password --region "$AWS_REGION" \
  | docker login --username AWS --password-stdin "$REGISTRY"
```

## Build the Four Images

These build contexts match this repository:

```bash
docker build -t etp-auth-service ./services/auth_service
docker build -t etp-order-service ./services/trade-api
docker build -t etp-executor-service ./services/executor
docker build -t etp-frontend ./frontend
```

For the Angular frontend, production endpoint values must be supplied through
its build configuration. The Kubernetes deployment guide explains the required
frontend URL setup.

## Tag and Push

```bash
for image in etp-auth-service etp-order-service etp-executor-service etp-frontend; do
  docker tag "$image:latest" "$REGISTRY/$image:latest"
  docker push "$REGISTRY/$image:latest"
done
```

For safer deployments, use a unique release tag instead of `latest`:

```bash
IMAGE_TAG="$(git rev-parse --short HEAD)"

for image in etp-auth-service etp-order-service etp-executor-service etp-frontend; do
  docker tag "$image:latest" "$REGISTRY/$image:$IMAGE_TAG"
  docker push "$REGISTRY/$image:$IMAGE_TAG"
done
```

Update the `images` section in [`k8s/base/kustomization.yaml`](k8s/base/kustomization.yaml)
to use the same registry and tag. Do not leave another AWS account ID or an old
image tag in that file.

## Verify the Push

```bash
for repo in etp-auth-service etp-order-service etp-executor-service etp-frontend; do
  aws ecr describe-images \
    --region "$AWS_REGION" \
    --repository-name "$repo" \
    --query 'imageDetails[?imageTags && contains(imageTags, `latest`)].{Digest:imageDigest,Pushed:imagePushedAt,Tags:imageTags}' \
    --output table
done
```

Verify Kubernetes will use the ECR images before applying:

```bash
kubectl kustomize k8s/overlays/four-lb \
  | grep -E 'image: .*ecr.*etp-(auth-service|order-service|executor-service|frontend)'
```

Apply the deployment after the image references are updated:

```bash
kubectl apply -k k8s/overlays/four-lb
kubectl rollout status deployment/auth-service --timeout=180s
kubectl rollout status deployment/order-service --timeout=180s
kubectl rollout status deployment/executor-service --timeout=180s
kubectl rollout status deployment/frontend --timeout=180s
```

## Other Projects

For another project, change the repository names, Docker build directories,
container ports, and the corresponding `images` entries in Kustomize. Never put
`.env`, passwords, API keys, JWT secrets, or Kubernetes `secrets.env` into Git.
