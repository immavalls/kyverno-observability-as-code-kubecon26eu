# Full Observability for Kyverno with Observability-as-Code

This repo is to reproduce the demo for KyvernoCON EU 2026 talk [Full Observability for Kyverno with Observability-as-Code](https://eyeveebee.dev/full-observability-for-kyverno-with-observability-as-code)

This is for demo-purposes only, it's not production-ready and does not apply best-practices. It's just to showcase how we can achieve Observability-as-Code in GitOps for Kyverno.

## Pre-requisites

- [Docker](https://docs.docker.com/engine/install/), [kind](https://kind.sigs.k8s.io/), [kubectl](https://kubernetes.io/docs/tasks/tools/) and [Helm](https://helm.sh/docs/intro/install/)
- [Git](https://git-scm.com/install/)
- (Optional) [Lens](https://lenshq.io/products/lens-k8s-ide)

## Setup

We'll pre-install most of this for the live demo, as we have limited time during the session.

### Create the k8s cluster and install ArgoCD

```bash
kind create cluster --name kyverno-demo

kubectl create namespace argocd
kubectl apply --server-side -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Server-side apply is needed because ArgoCD's CRDs are too large for client-side apply. See the [ArgoCD getting started guide](https://argo-cd.readthedocs.io/en/stable/getting_started/).

### Deploy Prometheus and Tempo

Both go in the `kyverno` namespace, which is where Kyverno and the Grafana datasources expect them (`prometheus-server.kyverno.svc:9090`, `tempo.kyverno.svc:3200` and `:4317`). They use the official Helm charts: [prometheus](https://github.com/prometheus-community/helm-charts/tree/main/charts/prometheus) (prometheus-community) and [tempo](https://github.com/grafana-community/helm-charts) (grafana-community).

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana-community https://grafana-community.github.io/helm-charts
helm repo update

helm install prometheus prometheus-community/prometheus -n kyverno --create-namespace -f - <<'EOF'
alertmanager: { enabled: false }
prometheus-pushgateway: { enabled: false }
prometheus-node-exporter: { enabled: false }
kube-state-metrics: { enabled: false }
server:
  service:
    servicePort: 9090
extraScrapeConfigs: |
  - job_name: kyverno
    kubernetes_sd_configs:
      - role: pod
        namespaces:
          names: [kyverno]
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_label_app_kubernetes_io_part_of]
        action: keep
        regex: kyverno
      - source_labels: [__address__]
        action: replace
        regex: ([^:]+)(?::\d+)?
        replacement: $1:8000
        target_label: __address__
      - source_labels: [__meta_kubernetes_pod_label_app_kubernetes_io_component]
        target_label: component
      - source_labels: [__meta_kubernetes_pod_name]
        target_label: pod
EOF

helm install tempo grafana-community/tempo -n kyverno
```

Tested with chart versions: prometheus 29.47.2 and tempo 3.2.1. The Kyverno docs ([monitoring](https://kyverno.io/docs/guides/monitoring/#setting-up-prometheus), [tracing](https://kyverno.io/docs/guides/tracing/#installation-and-setup)) describe the same setup with older instructions.

### Deploy Kyverno and Grafana on ArgoCD

- [kyverno-app.yaml](./infrastructure/kyverno-app.yaml)
- [kyverno-policies-app](./infrastructure/kyverno-policies-app.yaml)
- [grafana-app.yaml](./apps/grafana/grafana-app.yaml)
- [grafana-provision-app.yaml](./apps/grafana/grafana-provision-app.yaml): creates the dashboard and alert ConfigMaps from [apps/grafana/provision](./apps/grafana/provision), which the Grafana sidecars load

```bash
kubectl apply -f infrastructure/kyverno-app.yaml
kubectl apply -f infrastructure/kyverno-policies-app.yaml
kubectl apply -f apps/grafana/grafana-app.yaml
kubectl apply -f apps/grafana/grafana-provision-app.yaml
```

Grafana only reads alert rules at startup. If the `admission-fails` alert rule is missing, restart Grafana: `kubectl rollout restart deploy/grafana-standalone -n kyverno`.

After the install, port forward ArgoCD and Grafana:

```bash
kubectl port-forward -n kyverno service/grafana-standalone 3000:80
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

And open https://localhost:8080/applications for ArgoCD.

And http://localhost:3000/ for Grafana. User and password `admin`.

## Live demo

Follow the step-by-step [demo script](./demo-script.md).

## Clean-up

```bash
kind delete cluster --name kyverno-demo
```