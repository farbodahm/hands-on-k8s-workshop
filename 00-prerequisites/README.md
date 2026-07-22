# 00 - Prerequisites

Before touching Kubernetes, you need the tools installed and a mental model of
what the pieces are. This module sets both up.

## What Kubernetes is, in one paragraph

Kubernetes (often written "k8s", the 8 standing for the eight letters between k
and s) is a system that runs containers for you across one or more machines. You
tell it the desired state ("I want three copies of this container running, each
reachable on port 8080"), and it works to make reality match that state. If a
container crashes, it restarts it. If a machine dies, it moves the work
elsewhere. You describe what you want in YAML files and hand them to the cluster.

## Core vocabulary

You will meet these terms in module 01 and after. A short definition now makes
the rest easier.

- **Cluster**: the whole system, one or more machines managed together.
- **Node**: a single machine in the cluster. With k3d, each node is a Docker
  container. Nodes are either the control plane (the brain) or workers (where
  your containers run). K3s often combines both roles on one node.
- **Control plane**: the components that make global decisions and hold cluster
  state. The API server is the front door; you talk to it through `kubectl`.
- **Pod**: the smallest thing Kubernetes runs. A Pod wraps one container (or a
  few tightly coupled ones) and gives them a shared network address.
- **Container image**: the packaged application plus its dependencies. You build
  one with Docker in module 02.
- **YAML manifest**: a text file describing a resource you want in the cluster.

## Tools you need

- **Docker**: runs the containers. k3d needs it to host the cluster nodes.
- **k3d**: creates K3s clusters inside Docker.
- **kubectl**: the client you use to talk to the cluster.
- **helm**: a package manager for Kubernetes, used in module 11.
- **Go**: to build the sample application in module 02.

## Installing on macOS with Homebrew

```sh
# Docker: install Docker Desktop, or colima as a lighter alternative
brew install --cask docker      # then launch Docker Desktop once
# or:
brew install colima docker && colima start

brew install k3d
brew install kubectl            # skip if already present
brew install helm
brew install go                 # skip if already present
```

## kubectl quality-of-life setup

Add these to your shell profile (`~/.zshrc`). They make every later module
faster.

```sh
alias k=kubectl
source <(kubectl completion zsh)
complete -o default -F __start_kubectl k
```

The `kubectl` completion lets you tab-complete resource names, which matters
once Pod names have random suffixes.
