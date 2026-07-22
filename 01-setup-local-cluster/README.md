# 01 - Set up a local cluster

Now you create a real K3s cluster and learn to look inside it. This is where the
vocabulary from module 00 becomes concrete.

## What k3d does

`k3d cluster create` starts one or more Docker containers that act as your
cluster nodes, runs K3s inside them, and writes the connection details into your
kubeconfig. The kubeconfig (`~/.kube/config`) is the file kubectl reads to know
which cluster to talk to and how to authenticate.

A single command gives you a working control plane and worker in seconds.

## Creating a cluster

```sh
k3d cluster create learn --agents 1
```

This creates a cluster named `learn` with one server node (control plane) and
one agent node (worker). k3d also updates your kubeconfig and switches the
current context to the new cluster.

The `--agents 1` flag gives you a second node so you can later see how Pods
spread across nodes. You can create a cluster with more agents to simulate a
larger environment.

## Contexts

A context is a named pointer to a cluster plus a user and default namespace.
kubectl acts on the current context.

```sh
kubectl config get-contexts     # list contexts
kubectl config current-context  # show the active one
kubectl config use-context k3d-learn   # switch (k3d prefixes names with k3d-)
```

## Looking inside the cluster

```sh
kubectl cluster-info        # API server and core service addresses
kubectl get nodes           # list nodes and their status
kubectl get nodes -o wide   # add IPs, OS, container runtime versions
```

Nodes show a `Ready` status once K3s has finished starting.

## Namespaces

A namespace is a virtual partition inside the cluster. Resources live in a
namespace, and names only need to be unique within one. Kubernetes ships with a
few:

- `default`: where your resources go if you do not specify a namespace.
- `kube-system`: the cluster's own components (DNS, networking, and so on).

```sh
kubectl get namespaces
kubectl get pods -n kube-system   # see what K3s runs internally
```

You will meet namespaces properly in module 12. For now, know that `default` is
where your work lands.

## Managing the cluster with k3d

```sh
k3d cluster list            # list clusters
k3d cluster stop learn      # stop without deleting
k3d cluster start learn     # start again
k3d cluster delete learn    # remove entirely
```

Stopping frees resources without losing your work. Deleting wipes everything,
which is useful when you want a clean slate.
