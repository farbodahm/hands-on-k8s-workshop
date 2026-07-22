# Task 03 - Scale out and load-balance

## Goal

Run three replicas of the app and put a Service in front of them. Prove that
requests spread across the Pods.

## Expected outcome

- The `hello` Deployment runs 3 replicas.
- A ClusterIP Service named `hello` lists 3 endpoints.
- Repeated requests through the Service return more than one distinct Pod name.

## How to check

```sh
kubectl get deployment hello
# READY 3/3

kubectl get endpoints hello
# ENDPOINTS lists three IP:port pairs

kubectl run curl --rm -it --image=curlimages/curl --restart=Never -- \
  sh -c 'for i in $(seq 1 10); do curl -s hello; done'
# output includes at least two different "hello from ..." pod names
```

## Steps

1. Set `replicas: 3` in the module 02 `deployment.yaml` and apply it (or use
   `kubectl scale` for a quick test).
2. Write `service.yaml` as a ClusterIP Service selecting `app: hello`, port 80
   to targetPort 8080. Apply it.
3. Check `kubectl get endpoints hello` shows three backends.
4. Run the throwaway curl Pod and confirm the responses come from different
   Pods.
5. Scale to 1 replica, re-run the curl loop (one name only), then back to 3.
   Watch the endpoint list track the change.

## Hints

- If the curl loop always returns the same Pod name, check you have more than
  one Ready replica and that the Service selector matches the Pod labels exactly.
- `kubectl get endpoints` is the fastest way to see whether a Service actually
  found any Pods. An empty endpoint list almost always means a selector typo.
- The Service name `hello` is resolvable by DNS inside the cluster, which is why
  `curl -s hello` works from the throwaway Pod. It would not work from your Mac;
  that needs Ingress (module 04) or a port-forward.
- Try `kubectl get service hello -o yaml` and find the ClusterIP that was
  assigned. That IP is stable for the life of the Service even as Pods churn.
