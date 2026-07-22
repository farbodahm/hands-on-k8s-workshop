# Task 00 - Get the tools ready

## Goal

Install every tool the path depends on and confirm each one runs.

## Expected outcome

Every command below prints a version without error, and Docker is running.

## How to check

```sh
docker version         # both Client and Server sections print
docker run --rm hello-world   # prints a "Hello from Docker!" message
k3d version            # prints k3d and k3s versions
kubectl version --client   # prints a client version
helm version           # prints a version
go version             # prints a Go version
```

If `docker version` shows the Client but errors on the Server, Docker is not
running yet. Start Docker Desktop (or run `colima start`) and retry.

## Steps

1. Install Docker and start it. Verify with `docker run --rm hello-world`.
2. Install k3d, kubectl, helm, and Go (see README for the Homebrew commands).
   Some may already be present; check with the version commands first.
3. Add the `k` alias and kubectl completion to `~/.zshrc`, then open a new
   shell or run `source ~/.zshrc`.
4. Run every command in the "How to check" section and confirm all pass.

## Hints

- On Apple Silicon, colima is a lighter Docker runtime than Docker Desktop and
  avoids the desktop app. `colima start` boots it; `colima status` checks it.
- `hello-world` failing with a permission or socket error almost always means
  Docker is not running, not that it is misinstalled.
- You do not need a Kubernetes cluster yet. That is module 01. This module only
  proves the tools work.
