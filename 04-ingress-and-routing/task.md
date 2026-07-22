# Task 04 - Expose and route HTTP traffic

## Goal

Reach your app from your Mac's browser through Ingress, then route two paths (or
two hosts) to two different Services.

## Expected outcome

- `curl http://hello.localhost:8081/` from your Mac returns a pod name.
- A second Deployment/Service is reachable through a different path or host on
  the same Ingress.

## How to check

```sh
kubectl get ingress
# shows your ingress with the host(s) and the address

curl http://hello.localhost:8081/
# hello from hello-...

# after adding the second service and path:
curl http://hello.localhost:8081/v2
# a response from the second app
```

## Steps

1. Recreate the cluster with the port mapping `-p "8081:80@loadbalancer"` (see
   README). Re-import the image and reapply the Deployment and Service.
2. Write and apply `ingress.yaml` routing `hello.localhost` to the `hello`
   Service. Curl it from your Mac.
3. Build a second variant of the app so you can tell them apart. The simplest
   way: run a second Deployment from the same image but a different name, and a
   matching Service. (Or change the response text, rebuild as `hello-go:0.2`,
   and import it.)
4. Extend the Ingress with a second rule, either a `/v2` path or a
   `v2.localhost` host, pointing at the second Service.
5. Curl both routes and confirm each reaches the intended app.

## Hints

- If curl hangs or connection-refuses, the port mapping is missing. Check
  `docker ps` for the k3d load balancer container and confirm 8081 is published.
  The mapping can only be set at cluster-create time, so recreating is expected.
- `*.localhost` resolves to 127.0.0.1 on macOS, so you do not need to touch
  `/etc/hosts`. If you invent a name without the `.localhost` suffix, you do.
- `kubectl describe ingress hello` shows how Traefik parsed your rules and lists
  the backends. Use it when a route does not behave.
- With `pathType: Prefix` and a `/v2` path, requests to `/v2` and `/v2/anything`
  both match. Your Go app serves `/` for any path, so both routes return a
  response; the pod name tells you which Deployment answered.
