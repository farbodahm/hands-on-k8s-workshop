# Task 02 - Run the Go app in the cluster

## Goal

Build the Go app into an image, get it into the cluster, and run it first as a
bare Pod, then as a Deployment. See the Deployment self-heal.

## Expected outcome

- A Deployment named `hello` runs one Pod, status `Running`.
- `curl` through a port-forward returns `hello from <pod-name>`.
- Deleting the Pod causes a replacement to appear automatically.

## How to check

```sh
kubectl get deployment hello
# READY 1/1, AVAILABLE 1

kubectl get pods -l app=hello
# one pod, STATUS Running

kubectl port-forward deployment/hello 8080:8080 &
curl localhost:8080
# hello from hello-xxxxxxxxxx-yyyyy

# self-healing check:
kubectl delete pod -l app=hello
kubectl get pods -l app=hello -w
# a new pod appears and reaches Running; stop watching with Ctrl-C
```

## Steps

1. Create `main.go` and `Dockerfile` in this directory (see README). Run
   `go mod init hello-go` so the build has a module.
2. Build the image: `docker build -t hello-go:0.1 .`
3. Import it into the cluster: `k3d image import hello-go:0.1 -c learn`
4. Apply `pod.yaml`, inspect it with `describe` and `logs`, port-forward, and
   curl it. Then delete the bare Pod.
5. Apply `deployment.yaml`. Confirm the Deployment created a Pod.
6. Delete the Pod and watch the Deployment replace it. This is the difference
   between a bare Pod and a managed one.

## Hints

- Forgetting `go mod init` makes the Docker build fail. Run it in the directory
  with `main.go`.
- If a Pod is stuck in `ErrImagePull` or `ImagePullBackOff`, the image is not in
  the cluster. Re-run `k3d image import` and check the image name and tag match
  the manifest exactly.
- `imagePullPolicy: IfNotPresent` is required for locally imported images.
  Without it, Kubernetes tries to pull from a registry and fails.
- The Pod name has two random suffixes because the Deployment owns a ReplicaSet,
  which owns the Pods. Run `kubectl get replicasets` to see the middle layer.
