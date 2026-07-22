# 03 - Services and scaling

You have one Pod. Now you run several copies and put a stable address in front
of them that spreads traffic across all of them.

## Scaling a Deployment

Each running copy of a Pod is a replica. Change the replica count and Kubernetes
adds or removes Pods to match.

```sh
kubectl scale deployment hello --replicas=3
kubectl get pods -l app=hello -o wide
```

Better, set `replicas: 3` in `deployment.yaml` and `kubectl apply` it, so the
file stays the source of truth. Editing the file and applying is the habit to
build; the `scale` command is handy for quick experiments.

Notice in `-o wide` output that Pods may land on different nodes. The scheduler
decides placement.

## The problem Services solve

Pods are disposable. Each gets its own IP, and that IP changes every time the
Pod is replaced. You cannot hardcode a Pod IP. A Service gives you one stable
name and address that always points at the current set of healthy Pods.

## How a Service finds its Pods

A Service uses a label selector, the same mechanism the Deployment uses. Any Pod
matching the selector becomes a backend (an "endpoint") of the Service.

`service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: hello
spec:
  selector:
    app: hello
  ports:
    - port: 80
      targetPort: 8080
```

`port` is the port the Service listens on. `targetPort` is the container port it
forwards to. Here clients hit port 80 and the Service forwards to 8080 in the
Pod.

## Service types

- **ClusterIP** (the default): reachable only from inside the cluster. Perfect
  for one service talking to another.
- **NodePort**: opens a port on every node so you can reach the Service from
  outside. Useful locally for a quick external check.
- **LoadBalancer**: asks the environment for an external load balancer. K3d maps
  this to a port on your machine.

## DNS and service discovery

The cluster runs an internal DNS server (CoreDNS). Every Service gets a DNS name
of the form `<service>.<namespace>.svc.cluster.local`. Inside the cluster, one
Pod can reach the Service simply by the name `hello` (in the same namespace).
This is how your app will find the database in module 06.

## Load balancing

A ClusterIP Service load-balances across its endpoints. Send several requests
and they land on different Pods.

```sh
kubectl apply -f service.yaml
kubectl get service hello
kubectl get endpoints hello        # the Pod IPs behind the service

# reach it from inside the cluster:
kubectl run curl --rm -it --image=curlimages/curl --restart=Never -- \
  sh -c 'for i in 1 2 3 4 5 6; do curl -s hello; done'
# you should see more than one distinct pod name in the output
```

The `kubectl run` line starts a throwaway Pod that curls the Service six times,
then deletes itself. Because the requests spread across replicas, you see
different hostnames.

## Inspecting endpoints

`kubectl get endpoints hello` lists the Pod IPs currently behind the Service.
Scale the Deployment up and down and watch the endpoint list change. This is the
Service tracking healthy Pods in real time.
