# Kubernetes Production Platform

A production-oriented Kubernetes reference deployment for a stateless HTTP service.

## Included

- isolated namespace and application configuration
- rolling updates and graceful termination
- CPU and memory requests and limits
- liveness, readiness, and startup probes
- horizontal pod autoscaling
- pod disruption budget
- ClusterIP service and NGINX Ingress
- Kustomize-based deployment

## Prerequisites

- Kubernetes 1.27+
- kubectl
- metrics-server
- NGINX Ingress Controller

## Deploy

```bash
kubectl apply -k k8s/
kubectl -n platform get all
kubectl -n platform rollout status deployment/demo-api
```

## Test

```bash
curl -H 'Host: platform.local' http://INGRESS_ADDRESS/health
kubectl -n platform describe hpa demo-api
```

The manifests use a public demo container and contain no credentials. Replace the image and host before using the configuration outside a lab environment.
