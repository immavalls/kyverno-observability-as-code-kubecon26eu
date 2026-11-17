# Demo script

Step-by-step script for the live demo (about 12 minutes). Setup instructions are in the [README](./README.md).

> **Note:** this script follows the current setup (ArgoCD, Kyverno, Prometheus, Tempo, Grafana). The README may still be updated to match it.

## Before you start

```bash
kubectl config use-context kind-kyverno-demo
kubectl get pods -A                      # everything Running/Completed

# the namespace used by the demo pods
kubectl create namespace demo-app --dry-run=client -o yaml | kubectl apply -f -

# port-forwards (leave running)
kubectl port-forward -n kyverno svc/grafana-standalone 3000:80 &
kubectl port-forward svc/argocd-server -n argocd 8080:443 &

# clean slate
kubectl delete pod demo-pass demo-fail -n demo-app --ignore-not-found
```

Open:

- Lens, cluster `kind-kyverno-demo` (optional)
- ArgoCD <https://localhost:8080>
- Grafana <http://localhost:3000> (`admin` / `admin`)
- A terminal, and an editor on this repo

ArgoCD password:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
```

## Act 0: The cluster in Lens (~2 min)

1. **Overview**: a local kind cluster.
2. **Namespaces**: `argocd` (GitOps engine), `kyverno` (policy engine, plus Prometheus, Tempo and Grafana), `demo-app` (where the demo pods run).
3. **Workloads > Pods** (namespaces `kyverno` and `argocd`): the whole stack in one view.
4. **Custom Resources > kyverno.io > Cluster Policy**: where the Kyverno policies live.
5. **Custom Resources > wgpolicyk8s.io > Policy Report**: the audit results (used in Act 2).

## Act 1: GitOps with ArgoCD (~2 min)

1. In the ArgoCD UI show the applications: `kyverno`, `kyverno-policies`, `grafana-standalone` (Synced / Healthy).
2. Open `kyverno-policies` and show the policies it deploys from [infrastructure/kyverno-policies](./infrastructure/kyverno-policies).
3. Everything, including the Grafana dashboard and alerts, comes from this repo. That is the Observability-as-Code idea.

## Act 2: Policies in action (~3 min)

1. Show the policies:

   ```bash
   kubectl get cpol
   ```

   Open [check-demo-label.yaml](./infrastructure/kyverno-policies/check-demo-label.yaml).

2. A pod with the right label (admitted). Keep Lens on the `demo-app` pods to see it appear:

   ```bash
   kubectl run demo-pass --image=nginx:latest --labels="demo-approved=true" -n demo-app
   ```

3. A pod with the wrong label (denied):

   ```bash
   kubectl run demo-fail --image=nginx:latest --labels="demo-approved=false" -n demo-app
   ```

   Expected: `Nginx pods must have the label 'demo-approved: true' to be deployed.`

4. Audit mode (`check-team-label` reports instead of blocking):

   ```bash
   kubectl get policyreports -n demo-app
   ```

## Act 3: Metrics with Prometheus and Grafana (~2 min)

1. Open the **Kyverno Metrics** dashboard in Grafana: admission request rate, policy execution latency, policy results.
2. Generate activity and watch the panels move:

   ```bash
   for i in 1 2 3 4 5; do
     kubectl run demo-pass-$i --image=nginx:latest --labels="demo-approved=true" -n demo-app
   done
   ```

3. Show the alert rules provisioned from [kyverno-alerts.yaml](./apps/grafana/provision/kyverno-alerts.yaml).

## Act 4: Traces with Tempo (~3 min)

1. Grafana > **Explore** > datasource **Tempo** > TraceQL:

   ```
   {resource.service.name="kyverno-admission-controller"}
   ```

2. Open a trace: the admission request, then a `POLICY /<name>` span per policy with rule spans beneath.
3. Narrow to the denied pod's rule:

   ```
   {resource.service.name="kyverno-admission-controller" && name="RULE check-for-demo-label"}
   ```

   Click the span and show its attributes: resource name and namespace, operation, rule result and message.
4. Close: policy latency from traces and metrics can drive SLOs and error-budget alerts.

## Clean-up

```bash
kubectl delete pod -n demo-app --all
kind delete cluster --name kyverno-demo
```
