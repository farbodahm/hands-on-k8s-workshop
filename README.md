# Hands-On Kubernetes Workshop

I originally created this short, practical introduction to Kubernetes for a workshop. After receiving positive feedback from participants, I decided to publish it for others to use.

The goal is to help you gain hands-on Kubernetes experience as quickly as possible without requiring you to work through long, theory-heavy courses first.

This workshop covers:

* Setting up and using k3s
* Understanding the core concepts and terminology of Kubernetes
* Deploying an application
* Deploying and connecting a database
* Working with namespaces
* Building the foundational knowledge needed to start using Kubernetes confidently

> **Disclaimer:** Claude provided the overall course structure, while the topics, examples, and content were selected and designed by me.

---

# Kubernetes learning path

A hands-on path for learning Kubernetes on a local K3s cluster. You build one Go
application and grow it step by step: a single Pod, then many replicas behind a
load balancer, then external routing, config, a database, health checks,
autoscaling, and packaging.

Each numbered directory is one module. Work through them in order. Every module
has two files:

- `README.md` is the tutorial. It explains the concepts and the commands, with
  the terminology you need to know.
- `task.md` is the exercise. It states what to build, how to check that it
  works, and hints if you get stuck. Write the YAML and run the commands
  yourself. The README is there to lean on, not to copy from.

## Why K3s and k3d

K3s is a full Kubernetes distribution from Rancher, trimmed down to a single
small binary. It is real Kubernetes, certified by the CNCF, so what you learn
here transfers to any cluster.

K3s runs on Linux. On macOS you run it through k3d, a tool that starts K3s
inside Docker containers. Your cluster nodes are containers, which makes it fast
to create and destroy clusters while you experiment. Other local options exist
(minikube, kind, Docker Desktop's built-in Kubernetes), but k3d gives you actual
K3s with almost no overhead.

## The modules

| Module | What you learn |
|--------|----------------|
| 00 | Install the tools, understand cluster/node/Pod vocabulary |
| 01 | Create and inspect a K3s cluster with k3d |
| 02 | Containerize a Go app and run it as a Pod, then a Deployment |
| 03 | Scale to many replicas and load-balance with a Service |
| 04 | Route external HTTP traffic by host and path with Ingress |
| 05 | Supply configuration with ConfigMaps and Secrets |
| 06 | Run Postgres with persistent storage and connect the app to it |
| 07 | Probes, resource limits, rolling updates, and rollbacks |
| 08 | Autoscale on CPU with the Horizontal Pod Autoscaler |
| 09 | Run one-off Jobs and scheduled CronJobs |
| 10 | Read logs and events, use `kubectl top`, debug running Pods |
| 11 | Package the app as a Helm chart |
| 12 | Isolate work with Namespaces and control access with RBAC |
| 13 | Tear the cluster down and plan what to study next |

## How to work through a module

1. Read `README.md` in the module directory.
2. Open `task.md` and attempt the task. Keep the README open for reference.
3. Run the verification steps in `task.md`. If they pass, move on.
4. If you get stuck, use the hints at the bottom of `task.md`.

## A note on kubectl

`kubectl` is the command-line client for Kubernetes. You will use it constantly.
A few patterns show up everywhere, so learn them early:

- `kubectl get <resource>` lists resources (`pods`, `deployments`, `services`, ...).
- `kubectl describe <resource> <name>` shows full detail and recent events.
- `kubectl apply -f file.yaml` creates or updates resources from a file.
- `kubectl delete -f file.yaml` removes them.
- `kubectl logs <pod>` prints a Pod's logs.
- Add `-o wide` for more columns, `-o yaml` for the full object, `-w` to watch.

Set up the `k` alias and shell completion in module 00. It saves a lot of typing.
