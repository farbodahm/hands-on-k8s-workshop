# 02 - Your first application

You will build a small Go web server, package it as a container image, and run
it in the cluster. This is the application you grow through the rest of the path.

## The sample application

A minimal HTTP server that reports which Pod handled the request. The hostname
inside a Pod is the Pod's name, so once you run several copies you can see which
one answered. Keep it this simple; the Kubernetes concepts are the point, not
the Go.

`main.go`:

```go
package main

import (
	"fmt"
	"log"
	"net/http"
	"os"
)

func main() {
	http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
		host, _ := os.Hostname()
		fmt.Fprintf(w, "hello from %s\n", host)
	})
	http.HandleFunc("/healthz", func(w http.ResponseWriter, r *http.Request) {
		w.WriteHeader(http.StatusOK)
		fmt.Fprintln(w, "ok")
	})
	port := os.Getenv("PORT")
	if port == "" {
		port = "8080"
	}
	log.Printf("listening on :%s", port)
	log.Fatal(http.ListenAndServe(":"+port, nil))
}
```

The `/healthz` endpoint exists for module 07's health checks. The `PORT`
environment variable exists for module 05's config work.

## The image

`Dockerfile` (a multi-stage build keeps the final image tiny):

```dockerfile
FROM golang:1.22 AS build
WORKDIR /src
COPY . .
RUN CGO_ENABLED=0 go build -o /app .

FROM gcr.io/distroless/static-debian12
COPY --from=build /app /app
EXPOSE 8080
ENTRYPOINT ["/app"]
```

Build it with a name and tag:

```sh
docker build -t hello-go:0.1 .
```

## Getting the image into the cluster

Your cluster nodes are separate Docker containers, so they cannot see images in
your local Docker by default. You have two clean options.

Import the image into the cluster:

```sh
k3d image import hello-go:0.1 -c learn
```

This copies your locally built image into every node. Use it for images you
build yourself. Public images from registries like Docker Hub are pulled
automatically and need no import.

## Pods

A Pod is the smallest deployable unit. You rarely create Pods directly in
production, but doing it once shows what a Deployment manages for you.

`pod.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: hello
spec:
  containers:
    - name: hello
      image: hello-go:0.1
      imagePullPolicy: IfNotPresent
      ports:
        - containerPort: 8080
```

`imagePullPolicy: IfNotPresent` tells Kubernetes to use the imported image
instead of trying to pull it from a registry.

```sh
kubectl apply -f pod.yaml
kubectl get pods
kubectl describe pod hello        # events, image, state
kubectl logs hello                # the server's log output
```

To reach it from your machine without any networking setup yet, forward a local
port to the Pod:

```sh
kubectl port-forward pod/hello 8080:8080
# in another terminal:
curl localhost:8080
```

## Deployments

A Deployment manages a set of identical Pods for you. It recreates a Pod if it
dies, and it handles updates. This is how you run applications for real.

`deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello
spec:
  replicas: 1
  selector:
    matchLabels:
      app: hello
  template:
    metadata:
      labels:
        app: hello
    spec:
      containers:
        - name: hello
          image: hello-go:0.1
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 8080
```

Two important ideas here:

- **Labels and selectors**: the Pods carry the label `app: hello`, and the
  Deployment's `selector` finds its Pods by that label. This label mechanism is
  how almost everything in Kubernetes connects, including Services in module 03.
- **The template**: `spec.template` is the blueprint for each Pod. Change it and
  the Deployment rolls out new Pods.

```sh
kubectl apply -f deployment.yaml
kubectl get deployments
kubectl get pods -l app=hello      # filter by label
kubectl delete pod <pod-name>      # watch the Deployment recreate it
```

Delete the standalone Pod from earlier so it does not confuse you:

```sh
kubectl delete pod hello
```
