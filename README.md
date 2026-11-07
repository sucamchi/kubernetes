# SynergyChat on Kubernetes

Notes from the boot.dev "Learn Kubernetes" course. I deployed the
SynergyChat app (a small chat app made of a web, API, and crawler
service) on a local minikube cluster and exposed it through Envoy
Gateway using the Kubernetes Gateway API.

## What I used

- **minikube** with the Docker driver as the local cluster
- **kubectl** for everything
- **Envoy Gateway** for the Gateway API controller
- **metrics-server** minikube addon for `kubectl top`
- **bootdev CLI** to run the lesson tests
- Docker Hub images: `synergychat-web`, `synergychat-api`,
  `synergychat-crawler`, `synergychat-testcpu`, `synergychat-testram`

## What is deployed

| Area | Resources |
|------|-----------|
| Web | `synergychat-web` Deployment (3 replicas), `web-service`, `synergychat-web-configmap` |
| API | `synergychat-api` Deployment, `api-service`, `synergychat-api-configmap` |
| Crawler | `synergychat-crawler` Deployment (3 containers) in namespace `crawler`, `crawler-service`, `synergychat-crawler-configmap` |
| Routing | `app-gatewayclass`, `app-gateway`, `web-httproute`, `api-httproute` |
| Storage | `synergychat-api-pvc` (1Gi, ReadWriteOnce) |
| Testing | `synergychat-testcpu` (50m CPU limit), `synergychat-testram` (40Gi memory request, stays Pending) |

Traffic is routed by hostname:

- `synchat.internal` goes to `web-service`
- `synchatapi.internal` goes to `api-service`

The hostnames resolve to the Envoy `LoadBalancer` IP through
`/etc/hosts` and `minikube tunnel`.

## What I learned

- **Pods and Deployments**: how replicas work, rolling updates, and
  scaling both at the pod level (replicas) and inside a pod
  (multiple containers).
- **ConfigMaps**: passing config with `envFrom` and `configMapKeyRef`,
  and why changing a ConfigMap alone does not restart pods.
- **Services**: how selectors match pod labels. A wrong selector means
  the Service has no endpoints, which is easy to miss because
  `kubectl get svc` still looks healthy.
- **Gateway API**: a Gateway needs a GatewayClass, and HTTPRoutes
  attach to a Gateway and match on hostname and path.
- **Volumes**: `emptyDir` is shared by containers in the same pod but
  dies with the pod. PersistentVolumeClaims request storage and
  dynamically create PersistentVolumes, which survive pod restarts.
- **Namespaces**: names are only unique within a namespace, and moving
  a resource means creating it in the new namespace and deleting the
  old one.
- **Resource limits vs requests**: limits cap usage (CPU gets
  throttled, memory gets the pod killed), while requests tell the
  scheduler how much room a pod needs. A request larger than the node
  keeps the pod `Pending`.
- **Debugging**: `kubectl logs`, `describe` (read the Events),
  `exec`, `top`, and `kubectl proxy`.

## Useful commands

```bash
kubectl apply -f <file>.yaml
kubectl get pods -A
kubectl -n crawler get pods
kubectl top pod
kubectl describe pod <pod>
kubectl logs <pod> --all-containers
kubectl proxy --port=8001
```
