# 11 - Packaging with Helm

By now you have many YAML files, and the values inside them repeat (the image
tag, the replica count, the port). Helm packages a set of manifests into a chart
and lets you template the values, so you install and upgrade a whole application
with one command.

## What Helm is

Helm is a package manager for Kubernetes. A chart is a bundle of templated
manifests plus default values. A release is an installed instance of a chart in
your cluster. Helm tracks releases, so upgrades and rollbacks work at the whole-
application level, not one resource at a time.

## Chart layout

```sh
helm create hello-chart
```

This scaffolds a chart. The parts that matter:

```
hello-chart/
  Chart.yaml          # chart name, version, description
  values.yaml         # default values, the knobs users can turn
  templates/          # manifests with {{ }} placeholders
    deployment.yaml
    service.yaml
    _helpers.tpl      # reusable template snippets
```

The generated chart is a full working example. Read `templates/deployment.yaml`
to see how values are wired in.

## Templating

Templates pull values from `values.yaml` with `{{ .Values.<path> }}`.

`values.yaml`:

```yaml
replicaCount: 3
image:
  repository: hello-go
  tag: "0.4"
  pullPolicy: IfNotPresent
service:
  port: 80
  targetPort: 8080
```

`templates/deployment.yaml` (excerpt):

```yaml
spec:
  replicas: {{ .Values.replicaCount }}
  template:
    spec:
      containers:
        - name: hello
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
```

One chart now produces different deployments depending on the values you pass.

## Install, upgrade, roll back

```sh
helm install hello ./hello-chart               # create a release named hello
helm list                                      # list releases
helm upgrade hello ./hello-chart --set replicaCount=5   # change a value
helm history hello                             # revisions of the release
helm rollback hello 1                          # back to revision 1
helm uninstall hello                           # remove everything the chart made
```

`--set` overrides a value on the command line; `-f myvalues.yaml` overrides with
a file. This is how one chart serves dev, staging, and production with different
values.

## Previewing before applying

```sh
helm template hello ./hello-chart              # render manifests, print them, apply nothing
helm install hello ./hello-chart --dry-run --debug   # validate against the cluster without installing
```

`helm template` is the fastest way to see exactly what your values produce.

## Using charts other people wrote

Most real infrastructure installs from public charts. The pattern:

```sh
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
helm install my-pg bitnami/postgresql
```

You could replace the hand-written Postgres from module 06 with a chart like
this. Reading a mature chart's `values.yaml` teaches a lot about how production
services are configured.
