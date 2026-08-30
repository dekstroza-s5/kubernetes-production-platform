# Kubernetes Production Platform

A production-oriented Kubernetes reference deployment for a stateless HTTP service. The project demonstrates the baseline controls used when moving a service from a development namespace into a maintained cluster environment.

## Architecture

```text
Client -> NGINX Ingress -> ClusterIP Service -> 3+ application pods
                                                |
                                      ConfigMap, probes, resources
                                                |
                                 HPA and PodDisruptionBudget
```

Kustomize renders the namespace, application configuration, Deployment, Service, Ingress, HorizontalPodAutoscaler and PodDisruptionBudget as one deployable unit.

## Design decisions

- RollingUpdate uses `maxUnavailable: 0` so a rollout does not intentionally reduce available capacity.
- Readiness removes an unhealthy pod from service endpoints; liveness restarts a stuck container; startup gives slow initialization time to finish.
- CPU and memory requests make scheduling predictable and allow percentage-based HPA calculations.
- A disruption budget preserves at least two replicas during voluntary maintenance.
- Containers run without root privileges, drop Linux capabilities and use a read-only root filesystem.
- Configuration is separated from the image through a ConfigMap.
- The image tag is explicit instead of `latest`, making deployments reproducible.

## Prerequisites

- Kubernetes 1.27 or newer
- kubectl with access to the target cluster
- metrics-server for HPA metrics
- NGINX Ingress Controller
- a local DNS or hosts entry for `platform.local`

Check the cluster before deployment:

```bash
kubectl version
kubectl get nodes
kubectl top nodes
kubectl get ingressclass
```

## Deploy

```bash
kubectl kustomize k8s/ > /tmp/platform-rendered.yaml
kubectl apply --dry-run=server -f /tmp/platform-rendered.yaml
kubectl apply -k k8s/
kubectl -n platform rollout status deployment/demo-api --timeout=120s
```

Expected result:

```text
deployment "demo-api" successfully rolled out
NAME                        READY   STATUS    RESTARTS
pod/demo-api-...            1/1     Running   0
service/demo-api            ClusterIP
horizontalpodautoscaler/demo-api   3   10
```

## Verify

```bash
kubectl -n platform get deploy,pods,svc,ingress,hpa,pdb
kubectl -n platform describe deployment demo-api
kubectl -n platform get endpointslices -l kubernetes.io/service-name=demo-api
curl -H 'Host: platform.local' http://INGRESS_ADDRESS/
```

A healthy deployment has three ready replicas, three service endpoints and no warning events.

## Rollout example

Update the image and monitor the rollout:

```bash
kubectl -n platform set image deployment/demo-api api=mendhak/http-https-echo:35
kubectl -n platform rollout status deployment/demo-api
kubectl -n platform rollout history deployment/demo-api
```

Rollback procedure:

```bash
kubectl -n platform rollout undo deployment/demo-api
kubectl -n platform rollout status deployment/demo-api
```

## Autoscaling test

Generate requests from inside the cluster and watch the HPA:

```bash
kubectl -n platform run load --rm -it --image=busybox:1.37 --   sh -c 'while true; do wget -q -O- http://demo-api >/dev/null; done'
kubectl -n platform get hpa,pods -w
```

Scale-down is deliberately stabilized for five minutes to avoid replica flapping.

## Troubleshooting

```bash
kubectl -n platform get events --sort-by=.lastTimestamp
kubectl -n platform describe pod POD
kubectl -n platform logs POD --previous
kubectl -n platform auth can-i get configmaps
```

- `Pending`: check resources, taints and scheduling events.
- `ImagePullBackOff`: confirm the image name, registry and pull secret.
- `CrashLoopBackOff`: inspect current and previous logs plus container exit codes.
- no ingress response: verify the ingress class, controller logs, host header and Service endpoints.
- HPA shows `unknown`: confirm metrics-server and CPU requests.

## Cleanup

```bash
kubectl delete -k k8s/
```

The manifests contain no credentials. Secrets should be supplied through a dedicated secrets controller or the deployment platform.
