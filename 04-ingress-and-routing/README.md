# 04 - Ingress and routing

A ClusterIP Service is only reachable inside the cluster. Ingress is how you
expose HTTP services to the outside world and route requests by hostname or URL
path to different Services.

## The pieces

- **Ingress**: a set of routing rules ("send `/api` to service A, `/` to service
  B", or "send `app.local` to this service").
- **Ingress controller**: the component that actually implements those rules. It
  watches Ingress objects and configures a reverse proxy. K3s ships with Traefik
  as its controller, already running in `kube-system`, so you do not install
  anything.

An Ingress object without a controller does nothing. K3s gives you both.

## How k3d exposes the port

The Traefik controller listens on port 80 inside the cluster. To reach it from
your Mac, the cluster needs a port mapped to your host. The simplest path is to
recreate the cluster with a port mapping:

```sh
k3d cluster delete learn
k3d cluster create learn --agents 1 -p "8081:80@loadbalancer"
```

This maps host port 8081 to port 80 on the k3d load balancer, which forwards to
Traefik. After recreating, reapply your Deployment and Service from modules 02
and 03 (the imported image is gone too, so re-import it).

## A basic Ingress

`ingress.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: hello
spec:
  rules:
    - host: hello.localhost
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: hello
                port:
                  number: 80
```

This routes requests for the host `hello.localhost` to the `hello` Service on
port 80. `hello.localhost` resolves to 127.0.0.1 on macOS automatically, so no
`/etc/hosts` editing is needed.

```sh
kubectl apply -f ingress.yaml
kubectl get ingress
curl http://hello.localhost:8081/
# hello from hello-...
```

## Routing by path

Ingress can send different paths to different Services. Suppose you deploy a
second app (call it `hello-v2`) with its own Deployment and Service. You can
route by path:

```yaml
paths:
  - path: /v1
    pathType: Prefix
    backend:
      service:
        name: hello
        port:
          number: 80
  - path: /v2
    pathType: Prefix
    backend:
      service:
        name: hello-v2
        port:
          number: 80
```

## Routing by host

Alternatively, route by hostname, giving each app its own name:

```yaml
rules:
  - host: v1.localhost
    http: { paths: [ ... hello ... ] }
  - host: v2.localhost
    http: { paths: [ ... hello-v2 ... ] }
```

Host-based routing is how one cluster serves many sites. Path-based routing is
common for splitting one site into services (a frontend and an API, for
instance).

## Ingress versus Service load balancing

Do not confuse the two layers. The Service load-balances across the Pods of one
application. The Ingress routes an incoming HTTP request to the right Service in
the first place. A request flows: client to Ingress controller (Traefik) to
Service to one Pod.
