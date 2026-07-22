# CLAUDE.md

## What this repository is

A self-paced Kubernetes learning path. The owner is learning k8s hands-on by
building one Go application and growing it across numbered modules: single Pod,
many replicas behind a load balancer, external routing, config, a database,
health checks, autoscaling, scheduled jobs, observability, Helm packaging, and
RBAC.

This is a teaching repo, not an application to ship. The point is for the owner
to learn the terminology and the `kubectl` / `k3d` / `helm` commands by doing
the work themselves.

## Environment

- Runs locally on macOS. K3s is Linux-only, so the cluster runs through **k3d**
  (K3s inside Docker containers). This is real, CNCF-certified Kubernetes.
- Tools: `docker`, `kubectl`, `k3d`, `helm`, `go`. Docker must be running.
- The cluster is named `learn` and is meant to be disposable (recreate freely).
- The sample app is a small Go HTTP server that reports its Pod hostname, with a
  `/healthz` endpoint (probes) and a `/db` endpoint (added in module 06). It is
  built and tagged `hello-go:0.1` through `0.4`, imported into the cluster with
  `k3d image import`.

## Structure

Each numbered directory is one module with two files:

- `README.md` — the tutorial: concepts, commands, terminology.
- `task.md` — the exercise: what to build, "expected outcome + how to check"
  with runnable verification commands, and hints.

Modules 00–13 run in order. See the root `README.md` for the full table.

## How to help in a future session

- The owner does the tasks themselves. Default to explaining, reviewing their
  work, and unblocking — not writing the manifests or running the cluster
  commands for them, unless they ask.
- Keep the writing style plain. This repo was authored with the
  `avoid-ai-writing` skill; match that when editing docs (no em dashes, minimal
  bold, no filler, sentence-case headings).
- The database module uses Postgres. If asked to switch databases, module 06 is
  the place.
- When adding modules or content, follow the existing README + task.md pattern
  and the numbering scheme (`NN-topic-name`).

## Status

The full path (modules 00–13) is drafted. No code has been written or cluster
created yet; that is the owner's work as they progress through the modules.
