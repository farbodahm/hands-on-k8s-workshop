# Task 01 - Create and inspect a cluster

## Goal

Create a K3s cluster with k3d, confirm it is healthy, and explore what runs
inside it.

## Expected outcome

- A cluster named `learn` exists with two nodes, both `Ready`.
- Your current kubectl context points at that cluster.
- You can list the system Pods that K3s runs.

## How to check

```sh
k3d cluster list
# learn should appear with servers 1/1 and agents 1/1

kubectl config current-context
# should print k3d-learn

kubectl get nodes
# two nodes, both STATUS Ready

kubectl get pods -n kube-system
# several pods, most Running (coredns, traefik, metrics-server, and others)
```

## Steps

1. Create the cluster with one agent node (see README).
2. Confirm the context switched to `k3d-learn`.
3. List the nodes and wait until both report `Ready`.
4. Look at `cluster-info` and the `kube-system` Pods to see the built-in pieces.
5. Practice the lifecycle: stop the cluster, start it again, and confirm the
   nodes return to `Ready`. Do not delete it; later modules reuse this cluster.

## Hints

- If `kubectl get nodes` errors with a connection refused, the cluster may still
  be starting. Wait a few seconds and retry.
- k3d prefixes the context name with `k3d-`, so the cluster `learn` becomes the
  context `k3d-learn`.
- Traefik in `kube-system` is the Ingress controller K3s installs by default.
  You will use it in module 04.
- If you ever want a completely fresh start, `k3d cluster delete learn` then
  recreate. Nothing in later modules is hard to rebuild.
