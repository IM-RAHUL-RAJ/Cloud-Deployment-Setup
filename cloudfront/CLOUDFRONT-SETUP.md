# CloudFront Deployment Guide

This guide covers the changes required before placing CloudFront in front of
the frontend and API LoadBalancers.

## 1. Application and Kubernetes Changes

### Frontend API endpoints

The frontend must use the CloudFront origin for browser API calls:

```text
https://d15caqwwfnbey2.cloudfront.net/auth
https://d15caqwwfnbey2.cloudfront.net/api/v1
```

Do not leave production Angular endpoints pointing to `localhost` or old ELB
hostnames. Build the frontend with the deployment URLs used by the application.

### Public Kubernetes Services

The frontend, auth, and order Services must be internet-facing so CloudFront
can reach them:

```yaml
metadata:
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-scheme: internet-facing
```

If a Service was originally created as internal, changing the annotation may not
change the existing ELB. Recreate only that Service, then wait for its new DNS
name:

```bash
kubectl delete service frontend auth-service order-service --ignore-not-found
kubectl apply -k k8s/overlays/four-lb
kubectl get svc frontend auth-service order-service
```

Do not delete Deployments, PVCs, the cluster, or Secrets for this change.

### URL and CORS configuration

Update `k8s/overlays/four-lb/urls.env`:

```env
FRONTEND_URL=https://d15caqwwfnbey2.cloudfront.net
AUTH_SERVICE_URL=http://<AUTH_ELB_DNS>
ORDER_SERVICE_URL=http://<ORDER_ELB_DNS>
EXECUTOR_SERVICE_URL=http://<EXECUTOR_ELB_DNS>
CORS_ALLOWED_ORIGINS=https://d15caqwwfnbey2.cloudfront.net
```

For this application, `FRONTEND_URL` is also used by the auth service when it
creates the CORS response. Apply the overlay after changing it:

```bash
kubectl apply -k k8s/overlays/four-lb
kubectl rollout status deployment/auth-service --timeout=180s
```

### Frontend image rebuild

If the Angular bundle contains API URLs at build time, rebuild the frontend
image after changing the public URL values. Use the same build process as CI and
push a new ECR tag. Then deploy that tag through `JenkinsfileCD`.

Do not put passwords or API keys in the frontend build arguments.

## 2. CloudFront Configuration

Distribution used in the tested setup:

```text
Distribution ID: E3O1K96RM8KJF
CloudFront domain: d15caqwwfnbey2.cloudfront.net
```

### Wait for deployment

```bash
aws cloudfront get-distribution \
  --id E3O1K96RM8KJF \
  --query 'Distribution.{Status:Status,Domain:DomainName}' \
  --output table
```

Continue only when `Status` is `Deployed`.

### Add origins

In **CloudFront > Distribution > Origins**, configure these origins:

| Origin | Domain | Protocol | Port |
| --- | --- | --- | --- |
| Frontend | `<FRONTEND_ELB_DNS>` | HTTP only | 80 |
| Auth | `<AUTH_ELB_DNS>` | HTTP only | 3000 |
| Order | `<ORDER_ELB_DNS>` | HTTP only | 8081 |

Enter only the DNS hostname in the origin domain field. Do not include
`http://` or a port in the hostname field.

The frontend origin used by the tested distribution was:

```text
k8s-default-frontend-1436e350e9-3414f338d650daa4.elb.ap-south-1.amazonaws.com
```

### Default behavior

Keep the default behavior pointing to the frontend origin:

```text
Path: /
Origin: frontend
Viewer protocol: Redirect HTTP to HTTPS
```

### API behaviors

Create these ordered behaviors:

```text
/auth/*     -> auth origin
/api/v1/*   -> order origin
```

For both API behaviors configure:

```text
Allowed methods: GET, HEAD, OPTIONS, PUT, POST, PATCH, DELETE
Cache policy: CachingDisabled
Viewer protocol: Redirect HTTP to HTTPS
```

Forward these headers to the origins:

```text
Authorization
Content-Type
Origin
```

Forward query strings and cookies when authentication or application behavior
requires them. Never cache login, refresh, or authenticated API responses.

### Optional CLI inspection

```bash
aws cloudfront get-distribution \
  --id E3O1K96RM8KJF \
  --query 'Distribution.{Status:Status,Origin:DistributionConfig.Origins.Items[].{Id:Id,Domain:DomainName,Protocol:CustomOriginConfig.OriginProtocolPolicy},Behaviors:DistributionConfig.CacheBehaviors.Items[].{Path:PathPattern,Target:TargetOriginId}}' \
  --output json
```

## 3. Cache Invalidation

After changing frontend assets or API behaviors:

```bash
aws cloudfront create-invalidation \
  --distribution-id E3O1K96RM8KJF \
  --paths '/*'
```

Wait for the invalidation to complete before testing from a browser.

## 4. Verification

### Frontend

```bash
curl -i https://d15caqwwfnbey2.cloudfront.net/
```

Expected result:

```text
HTTP/2 200
```

### Auth API

```bash
curl -i https://d15caqwwfnbey2.cloudfront.net/auth/public-key
```

Expected result:

```text
HTTP/2 200
Content-Type: application/json
```

### Login CORS preflight

```bash
curl -i -X OPTIONS \
  https://d15caqwwfnbey2.cloudfront.net/auth/login \
  -H 'Origin: https://d15caqwwfnbey2.cloudfront.net' \
  -H 'Access-Control-Request-Method: POST' \
  -H 'Access-Control-Request-Headers: content-type'
```

Expected result:

```text
HTTP/2 204
Access-Control-Allow-Origin: https://d15caqwwfnbey2.cloudfront.net
```

### Order API

```bash
curl -i https://d15caqwwfnbey2.cloudfront.net/api/v1/health
```

A `401` response can be correct when the endpoint requires authentication. A
CloudFront `502` usually means the origin is private, the origin protocol is
wrong, or the origin hostname is invalid.

## 5. Common Failures

- **CloudFront 502 with origin resolution error:** the ELB is private or the
  origin hostname is wrong. Use an internet-facing ELB.
- **CloudFront 502 with HTTPS origin:** set the custom origin protocol to
  `HTTP only` when the ELB serves plain HTTP.
- **CORS allows the old frontend ELB:** set both `FRONTEND_URL` and
  `CORS_ALLOWED_ORIGINS` to the CloudFront URL, then apply the Kubernetes overlay.
- **Frontend still calls an old ELB:** rebuild the frontend image with the
  CloudFront API base URLs and deploy the new ECR tag.
- **Order endpoint returns 401:** routing works; provide a valid access token.

Never commit `.env`, `k8s/base/secrets.env`, passwords, API keys, JWT secrets, or
private certificates.
