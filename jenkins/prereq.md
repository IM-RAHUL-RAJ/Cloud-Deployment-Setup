# Kubernetes Cleanup Prerequisite

Run these commands before Jenkins CD if old application workloads are still
running. Use the existing kubectl context; no AWS configuration or namespace
argument is required.

```bash
kubectl delete deployment \
  auth-service \
  order-service \
  executor-service \
  frontend \
  --ignore-not-found

kubectl delete statefulset kafka --ignore-not-found

kubectl get pods
```

This removes application workloads only. It does not delete the EKS cluster,
worker nodes, Services, Secrets, ConfigMaps, PVCs, or Kafka data.
