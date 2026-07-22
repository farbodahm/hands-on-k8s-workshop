# 08 - Autoscaling

You scaled by hand in module 03. Here Kubernetes scales for you based on load,
using the Horizontal Pod Autoscaler (HPA).

## Horizontal versus vertical

- **Horizontal scaling**: add or remove Pods. This is what the HPA does and what
  most web apps want.
- **Vertical scaling**: give a Pod more CPU or memory. Handled by the Vertical
  Pod Autoscaler, out of scope here.

## How the HPA works

The HPA watches a metric (commonly CPU usage) across a Deployment's Pods and
adjusts the replica count to keep the metric near a target. If you target 50%
CPU and Pods are averaging 80%, it adds replicas; when load drops, it removes
them.

It needs two things:

- **Resource requests** on the container, because CPU usage is measured as a
  percentage of the request. You added these in module 07.
- **metrics-server**, the component that collects CPU and memory usage. K3s
  installs it by default; confirm with `kubectl top`.

```sh
kubectl top nodes      # per-node CPU and memory
kubectl top pods       # per-pod usage; needs metrics-server running
```

If `kubectl top` errors, metrics-server is not ready yet. Wait, or check its Pod
in `kube-system`.

## Creating an HPA

```sh
kubectl autoscale deployment hello --cpu-percent=50 --min=1 --max=5
```

Or as a manifest, `hpa.yaml`:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: hello
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: hello
  minReplicas: 1
  maxReplicas: 5
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 50
```

```sh
kubectl apply -f hpa.yaml
kubectl get hpa
kubectl get hpa hello -w      # watch TARGETS and REPLICAS change under load
```

## Generating load

Drive CPU up so the HPA reacts. A busy request loop from inside the cluster
works:

```sh
kubectl run load --rm -it --image=curlimages/curl --restart=Never -- \
  sh -c 'while true; do wget -q -O- http://hello; done'
```

Your `/` handler is cheap, so you may need several load generators, or add a
deliberately CPU-heavy endpoint to the app for the demo. Watch replicas climb
toward `max`, then stop the load and watch them scale back down after the
cooldown period (a few minutes by default).

## Reading the HPA

`kubectl describe hpa hello` shows the current metric, the target, the current
and desired replica counts, and recent scaling events. It is the first place to
look when scaling does not behave.
