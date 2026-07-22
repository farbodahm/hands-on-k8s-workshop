# 10 - Observability and debugging

When something misbehaves, you need to see what the cluster sees. This module is
the toolkit for inspecting, debugging, and understanding running workloads. You
have used pieces of it already; here it comes together.

## The four questions and their commands

**What is the state?** `kubectl get`

```sh
kubectl get pods -o wide             # status, restarts, node, IP
kubectl get all                      # pods, services, deployments in the namespace
kubectl get events --sort-by=.lastTimestamp   # recent cluster events
```

**Why is it in that state?** `kubectl describe`

```sh
kubectl describe pod <pod>
```

The Events section at the bottom is the single most useful debugging output in
Kubernetes. Scheduling failures, image pull errors, probe failures, and
OOMKills all show up here.

**What is the app saying?** `kubectl logs`

```sh
kubectl logs <pod>                   # current logs
kubectl logs <pod> -f                # follow, like tail -f
kubectl logs <pod> --previous        # logs from the last crashed container
kubectl logs -l app=hello --tail=20  # combined logs from all matching pods
```

`--previous` is how you read why a container crashed, since the crashed
container is gone but its logs are kept.

**What is it using?** `kubectl top`

```sh
kubectl top nodes
kubectl top pods
```

## Getting inside a container

```sh
kubectl exec -it <pod> -- sh          # a shell in the container
kubectl exec <pod> -- env             # dump environment variables
kubectl exec <pod> -- cat /etc/hello/PORT   # read a mounted config file
```

Your distroless app image has no shell, so `exec ... sh` fails on it. That is a
deliberate security choice. For those cases, use ephemeral debug containers:

```sh
kubectl debug -it <pod> --image=busybox --target=hello
```

This attaches a temporary container with a shell into the running Pod's
namespaces, so you can inspect the network and processes without a shell in the
app image.

## Port-forwarding for direct access

```sh
kubectl port-forward deployment/hello 8080:8080
kubectl port-forward service/hello 9000:80
```

Port-forward bypasses Ingress and Services to reach a specific target, which
isolates whether a problem is in the app or in the routing.

## Reading resource relationships

```sh
kubectl get pod <pod> -o yaml         # the full object, including ownerReferences
kubectl explain deployment.spec.strategy   # docs for any field, offline
```

`kubectl explain` is an underused way to learn the schema without a browser.

## A debugging routine

When a Pod is not doing what you expect, work top-down: `get pods` to see the
status, `describe pod` to read the events, `logs` (add `--previous` if it
restarted) to read the app's own words, then `exec` or `debug` if you need to
poke around inside. Most problems are explained by the events or the logs before
you ever open a shell.
