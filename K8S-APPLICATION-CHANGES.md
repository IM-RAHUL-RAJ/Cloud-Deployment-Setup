# Application Changes Required for the Kubernetes Bundle

The `k8s/` folder provides the deployment infrastructure, but it cannot know the
names, ports, environment variables, health endpoints, or image repositories of
a different application. Before applying it to another project, complete this
checklist.

## 1. ECR Image URLs

The ECR image URLs are configured in:

```text
k8s/base/kustomization.yaml
```

Update the `images` section:

```yaml
images:
  - name: etp-auth-service
    newName: ACCOUNT_ID.dkr.ecr.REGION.amazonaws.com/team-auth
    newTag: v1
  - name: etp-order-service
    newName: ACCOUNT_ID.dkr.ecr.REGION.amazonaws.com/team-order
    newTag: v1
  - name: etp-executor-service
    newName: ACCOUNT_ID.dkr.ecr.REGION.amazonaws.com/team-executor
    newTag: v1
  - name: etp-frontend
    newName: ACCOUNT_ID.dkr.ecr.REGION.amazonaws.com/team-frontend
    newTag: v1
```

The `name` value must match the image name in the Deployment files. The
`newName` is the full ECR image URL. The `newTag` must be the tag that was
actually pushed.

You can verify the rendered image URLs with:

```bash
kubectl kustomize k8s/overlays/four-lb \
  | grep -E 'image: .*ecr.*'
```

Do not hard-code another team's AWS account ID or use `latest` for production.
Prefer a Git commit SHA or release tag.

## 2. Build Contexts and Local Image Names

The standard Deployment Kit expects these application folders:

| Workload | Default source folder | Local image name | Container port |
| --- | --- | --- | --- |
| Auth | `services/auth_service` | `auth-service` | `3000` |
| Order/trade API | `services/trade-api` | `order-service` | `8081` |
| Executor | `services/executor` | `executor-service` | `8082` |
| Frontend | `frontend` | `frontend` | `80` |

If a project uses different folders, update the Docker build commands and the
`*_SERVICE_DIR` values used by its Compose/build script. If it uses different
local image names, update the ECR tagging map before pushing.

Each Dockerfile must:

- Build a runnable image.
- Listen on the port declared in its Kubernetes Deployment.
- Read configuration from environment variables or `/config/application.yml`.
- Run as the non-root user expected by the Deployment security context.
- Expose a health endpoint or TCP port compatible with its probes.

## 3. Non-Secret Configuration

Edit:

```text
k8s/base/config.env
```

Replace the values for:

```env
DB_HOST=
DB_PORT=5432
DB_NAME=
DB_USER=
DB_SCHEMA_TRADE=
DB_SCHEMA_AUTH=
KAFKA_BOOTSTRAP_SERVERS=kafka:29092
AUTH_SERVICE_PORT=3000
ORDER_SERVICE_PORT=8081
EXECUTOR_SERVICE_PORT=8082
FRONTEND_PORT=80
FAUXNANCE_BASE_URL=
GEMINI_BASE_URL=
GEMINI_MODEL=
```

Add every non-secret environment variable read by the application. Typical
examples include feature flags, Kafka topic names, polling intervals, email
settings, and external API URLs.

Do not put passwords, tokens, API keys, or private keys in `config.env`.

## 4. Kubernetes Secrets

Copy the template locally:

```bash
cp k8s/base/secrets.env.example k8s/base/secrets.env
```

Add every secret key read by the application, for example:

```env
DB_PASSWORD=
JWT_SECRET=
FAUXNANCE_API_KEY=
GEMINI_API_KEY=
SMTP_USER=
SMTP_PASS=
```

The names must exactly match the names read by the code. The populated
`secrets.env` must remain ignored and must never be committed.

Kustomize converts this file into the generated `etp-secrets-*` Kubernetes
Secret. Application Deployments receive those values through `envFrom`.

## 5. Shared Application Configuration

Replace:

```text
k8s/base/generated/application.yml
```

This file is mounted into application containers at:

```text
/config/application.yml
```

The Deployment sets:

```text
SHARED_CONFIG=/config/application.yml
```

Keep the placeholders aligned with the environment variables in `config.env`
and `secrets.env`, for example:

```yaml
database:
  host: ${DB_HOST}
  port: ${DB_PORT:5432}
  name: ${DB_NAME}
  user: ${DB_USER}
  password: ${DB_PASSWORD}
```

`config.env` is a flat environment-variable contract. `application.yml` is a
structured file-based contract for services that load shared configuration.
They are complementary and must describe the same application settings.

## 6. Application Code Contracts

Before deployment, confirm the code uses the same contracts as the manifests.

### Auth service

The auth service must:

- Read `DB_HOST`, `DB_PORT`, `DB_NAME`, `DB_USER`, and `DB_PASSWORD`.
- Read `DB_SCHEMA_AUTH` if it uses a separate auth schema.
- Read `JWT_SECRET` and the matching issuer/expiry settings.
- Listen on `AUTH_SERVICE_PORT`.
- Expose the health path configured in `auth-service.yaml`.

### Order service

The order service must:

- Read the database and `DB_SCHEMA_TRADE` values.
- Read `JWT_SECRET` using the same signing configuration as auth.
- Read `KAFKA_BOOTSTRAP_SERVERS` and the same topic names as Kafka.
- Listen on `ORDER_SERVICE_PORT`.
- Expose the health path configured in `order-service.yaml`.
- Accept the request field types sent by the frontend. UUID fields must receive
  standard UUID values, not timestamps or arbitrary random strings.

### Executor service

The executor must:

- Read the same database, JWT, and Kafka contracts.
- Use the internal Kafka address `kafka:29092`.
- Listen on `EXECUTOR_SERVICE_PORT` if its probe expects a TCP port.
- Run only the intended number of polling replicas.

### Frontend

The frontend must:

- Build with `AUTH_SERVICE_URL`, `AUTH_SERVICE_PORT`, `ORDER_SERVICE_URL`, and
  `ORDER_SERVICE_PORT`.
- Generate API endpoint configuration from those values.
- Avoid hard-coded `localhost` URLs in production bundles.
- Use the actual public URLs from the selected overlay.
- Generate UUID request/idempotency values when the backend requires UUIDs.
- Include the frontend origin in the backend CORS allow list.

The browser's `localhost` means the user's own computer, not the EKS cluster.

## 7. Service and Probe Alignment

If ports or paths differ, update all matching locations:

- `config.env`
- The container port in the workload Deployment
- The Kubernetes Service port and target port
- Readiness, liveness, and startup probes
- `overlays/ingress/ingress.yaml` paths and backend ports
- Frontend build configuration
- Backend CORS configuration

A service can be running while still being removed from LoadBalancer endpoints if
its readiness probe points to the wrong path or port.

## 8. Public URLs

After `LoadBalancer` Services receive DNS names, update:

```text
k8s/overlays/four-lb/urls.env
```

Use host-only values when the application appends its service port itself. For
example:

```env
AUTH_SERVICE_URL=http://auth-load-balancer.example.com
ORDER_SERVICE_URL=http://order-load-balancer.example.com
EXECUTOR_SERVICE_URL=http://executor-load-balancer.example.com
FRONTEND_URL=http://frontend-load-balancer.example.com
CORS_ALLOWED_ORIGINS=http://frontend-load-balancer.example.com
```

Do not leave `lb-pending.invalid`, `localhost`, or another project's DNS names.
For production, prefer the one-ALB ingress overlay and HTTPS with ACM.

## 9. Namespace and Overlay Choice

The base creates an `etp` Namespace, but workloads must explicitly target the
intended namespace. Before deploying a new project, confirm the namespace
behavior in the manifests and use the same namespace in all `kubectl` commands.

Choose one overlay:

- `overlays/four-lb`: initial test with four public LoadBalancer Services.
- `overlays/ingress`: preferred browser deployment with one ALB and path routing.

Do not include `optional/postgres` when using an external RDS instance.

## 10. Pre-Apply Checklist

```bash
kubectl kustomize k8s/overlays/four-lb >/tmp/rendered.yaml
kubectl apply --dry-run=server -k k8s/overlays/four-lb

grep -E 'image: .*ecr' /tmp/rendered.yaml
grep -E 'DB_HOST|DB_NAME|KAFKA_BOOTSTRAP_SERVERS' k8s/base/config.env

test ! -e k8s/base/secrets.env || echo 'secrets.env is local only'
```

Only apply after image URLs, ports, probes, URLs, configuration keys, and
secret keys match the target project's code.
