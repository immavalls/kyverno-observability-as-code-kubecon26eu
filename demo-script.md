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

## Act 3: Metrics and alerts with Prometheus and Grafana (~3 min)

1. Open the [Kyverno Metrics dashboard](http://localhost:3000/d/Rg8lWBG7k/kyverno-metrics?orgId=1&from=now-1h&to=now&timezone=browser&var-DS_PROMETHEUS_KYVERNO=kyverno-prometheus-ds&var-cluster=$__all) in Grafana: admission request rate, policy execution latency, policy results.
2. Generate activity, passes and fails, and watch the panels move. **Start this first thing in Act 3**: the `admission-fails` alert needs a sustained burst of failures and takes about 2 to 3 minutes to fire.

   ```bash
   for i in $(seq 1 5); do
     kubectl run demo-pass-$i --image=nginx:latest --labels="demo-approved=true" -n demo-app
   done
   for i in $(seq 1 30); do
     kubectl run demo-fail-$i --image=nginx:latest --labels="demo-approved=false" -n demo-app
   done
   ```

   The 30 `demo-fail` requests are all denied, so no pods are left behind.

3. Open the [admission-fails alert rule](http://localhost:3000/alerting/grafana/cfgtyqqnwa134e/view) (all rules: <http://localhost:3000/alerting/list>). It goes **Normal, then Pending, then Firing** once the denied requests pass the threshold (mean of the failed results over 5 minutes above 2). The rule comes from [kyverno-alerts.yaml](./apps/grafana/provision/kyverno-alerts.yaml).

## Act 4: Traces with Tempo (~3 min)

1. Open [Grafana Explore with Tempo](http://localhost:3000/explore?schemaVersion=1&orgId=1&panes=%7B%22a%22%3A%7B%22datasource%22%3A%22tempo-datasource%22%2C%22queries%22%3A%5B%7B%22refId%22%3A%22A%22%2C%22datasource%22%3A%7B%22type%22%3A%22tempo%22%2C%22uid%22%3A%22tempo-datasource%22%7D%2C%22queryType%22%3A%22traceql%22%2C%22query%22%3A%22%7Bresource.service.name%3D%5C%22kyverno-admission-controller%5C%22%7D%22%7D%5D%2C%22range%22%3A%7B%22from%22%3A%22now-15m%22%2C%22to%22%3A%22now%22%7D%7D%7D) (datasource **Tempo**, TraceQL, last 15 minutes) and run:

   ```
   {resource.service.name="kyverno-admission-controller"}
   ```

2. Open a trace: the admission request, then a `POLICY /<name>` span per policy with rule spans beneath.
3. Narrow to the denied pod's rule, or [open it directly](http://localhost:3000/explore?schemaVersion=1&orgId=1&panes=%7B%22a%22%3A%7B%22datasource%22%3A%22tempo-datasource%22%2C%22queries%22%3A%5B%7B%22refId%22%3A%22A%22%2C%22datasource%22%3A%7B%22type%22%3A%22tempo%22%2C%22uid%22%3A%22tempo-datasource%22%7D%2C%22queryType%22%3A%22traceql%22%2C%22query%22%3A%22%7Bresource.service.name%3D%5C%22kyverno-admission-controller%5C%22%20%26%26%20name%3D%5C%22RULE%20check-for-demo-label%5C%22%7D%22%7D%5D%2C%22range%22%3A%7B%22from%22%3A%22now-15m%22%2C%22to%22%3A%22now%22%7D%7D%7D):

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
