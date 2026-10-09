# Kube Prometheus Stack on EKS

Install the kube-prometheus-stack Helm chart on an EKS cluster, deploy two demo apps, watch their CPU and memory in Grafana, and set up an alert that sends a notification to Discord. Everything after the install is done from the Grafana UI.

Chart: `prometheus-community/kube-prometheus-stack` version **92.2.0**.

## 1. What gets installed

| Component | What it does |
|---|---|
| Prometheus Operator | Manages Prometheus and Alertmanager from Kubernetes resources |
| Prometheus | Scrapes and stores metrics |
| Alertmanager | Groups and routes Prometheus alerts |
| Grafana | Dashboards and (in this guide) alerting |
| kube-state-metrics | Metrics about Kubernetes objects (deployments, pods, restarts) |
| node-exporter | Metrics about each node (one pod per node) |

## 2. Prerequisites

- An EKS cluster and `aws`, `kubectl` and `helm` (v3+) installed
- AWS credentials configured on the machine

## 3. Connect to the cluster

```bash
aws eks update-kubeconfig --name EKS --region us-east-1
kubectl get nodes
```

Replace the cluster name and region with yours.

## 4. Install the chart

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm search repo prometheus-community/kube-prometheus-stack     # confirm it shows 92.2.0

helm install kps prometheus-community/kube-prometheus-stack \
  --version 92.2.0 \
  --namespace monitoring --create-namespace
```

This is a default install, with no values file. On EKS that has two side effects worth knowing about:
- AWS manages the control plane, so etcd, scheduler, controller-manager and kube-proxy cannot be scraped. Their targets are missing or down, and a few built-in alerts about them may fire.
- Prometheus only picks up ServiceMonitors that carry the label `release: kps`. Not needed for this guide, but it matters when you add monitoring for an app's own metrics later.

## 5. Verify

```bash
kubectl get all -n monitoring
```

You should see these pods `Running`: alertmanager, grafana (3/3), the operator, kube-state-metrics, one node-exporter per node, and the prometheus pod.

## 6. Open the UIs

Get the Grafana admin password (username: `admin`):

```bash
kubectl -n monitoring get secret kps-grafana -o jsonpath='{.data.admin-password}' | base64 -d; echo
```

Port-forward (one terminal each):

```bash
kubectl -n monitoring port-forward svc/kps-grafana 3000:80
kubectl -n monitoring port-forward svc/kps-kube-prometheus-stack-prometheus 9090:9090
kubectl -n monitoring port-forward svc/kps-kube-prometheus-stack-alertmanager 9093:9093
```

Open `http://localhost:3000` (Grafana), `:9090` (Prometheus), `:9093` (Alertmanager). If `kubectl` runs on a remote machine, add `--address 0.0.0.0` or use an SSH tunnel.

## 7. Deploy two demo apps

```bash
kubectl create namespace demo
```

Save as `apps.yaml`:

- `web`: a quiet nginx web server (2 replicas), the low-usage baseline
- `stress`: burns CPU and holds memory on purpose, so the graphs have something to show. It has a CPU limit of 250m, so it runs at its limit.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
  namespace: demo
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
        - name: nginx
          image: nginx:1.27-alpine
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: 50m
              memory: 32Mi
            limits:
              cpu: 100m
              memory: 128Mi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: stress
  namespace: demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: stress
  template:
    metadata:
      labels:
        app: stress
    spec:
      containers:
        - name: stress
          image: polinux/stress
          command: ["stress"]
          args: ["--cpu", "1", "--vm", "1", "--vm-bytes", "100M", "--vm-hang", "1"]
          resources:
            requests:
              cpu: 100m
              memory: 100Mi
            limits:
              cpu: 250m
              memory: 200Mi
```

```bash
kubectl apply -f apps.yaml
kubectl -n demo get pods          # 2 web pods + 1 stress pod, all Running
```

## 8. See CPU and memory in Grafana (UI only)

Wait 1-2 minutes for the first metrics, then in Grafana go to **Dashboards**:

- **Kubernetes / Compute Resources / Namespace (Pods)**: set the namespace to `demo`. The `stress` pod sits at about 0.25 CPU (its limit) and about 100 MiB of memory; the `web` pods are nearly idle. Usage is shown next to requests and limits.
- **Kubernetes / Compute Resources / Pod**: pick one pod; includes a CPU throttling panel.
- **Kubernetes / Compute Resources / Workload**: the same data grouped by Deployment.
- **Node Exporter / Nodes**: CPU, memory, disk and network of the EC2 nodes.

## 9. Alerting from the Grafana UI

These are Grafana-managed alerts (Alerting menu in Grafana), separate from the Prometheus/Alertmanager alerts that ship with the chart. Menu names can differ slightly between Grafana versions.

### 9.1 Create a Discord webhook

1. In Discord, pick a server and channel (for example `#alerts`).
2. Channel gear icon (Edit Channel), then **Integrations**, **Webhooks**, **New Webhook**, **Copy Webhook URL**.

Treat the URL like a password; do not commit it or share it. (Slack works the same way: choose Slack as the integration and paste an incoming webhook URL.)

### 9.2 Add a contact point

1. **Alerting**, **Contact points**, **+ Add contact point**.
2. Name `discord-demo`, integration **Discord**, paste the webhook URL.
3. Click **Test**, then **Send test notification**: a message appears in the channel.
4. **Save contact point**.

### 9.3 Send alerts there, and make it fast

1. **Alerting**, **Notification policies**, edit the **Default policy**.
2. **Default contact point**: `discord-demo`.
3. **Timing options**: Group wait `10s`, Group interval `30s` (defaults are slower, which makes the Resolved message late in a demo).

### 9.4 Create the CPU alert rule

**Alerting**, **Alert rules**, **+ New alert rule**. Name: `CPU Usage`.

**Query** (data source: Prometheus, use the **Code** view).

To simply *see* CPU of every pod in `demo`, in cores:

```promql
sum by (pod) (rate(container_cpu_usage_seconds_total{namespace="demo", container!=""}[2m]))
```

For the **alert**, use the percentage of each pod's CPU limit, so the threshold is a percent:

```promql
100 * sum by (pod) (rate(container_cpu_usage_seconds_total{namespace="demo", container!=""}[2m]))
    / sum by (pod) (kube_pod_container_resource_limits{namespace="demo", resource="cpu"})
```

- `rate(...[2m])` turns the ever-growing CPU counter into "cores used right now".
- `container!=""` drops a duplicate pod-level series.
- `sum by (pod)` gives one series per pod, so each pod gets its own alert.
- `stress` reads about 100 (it runs at its limit); `web` pods read close to 0.

**Condition:** the alert condition is **Is above 80**. If you cannot see the threshold, turn on **Advanced options** in the query section to reveal the **Reduce** (Last) and **Threshold** expressions.

**Folder and evaluation group:**
- Folder: any folder you create (for example `Workshop`).
- Evaluation group: **+ New evaluation group**, name `demo`, interval `1m`. A group is a set of rules evaluated together on one schedule.
- Pending period: `0s` for a fast demo (in real life `2m` to `5m` to ignore short spikes).

**Contact point:** `discord-demo`. Save the rule.

### 9.5 Watch it fire

Under **Alerting**, **Alert rules** the state goes Normal, then Firing within about a minute, and the Discord message arrives. A typical message contains the value (`A=89.03` is the CPU percentage, `B=1` is the threshold check result), the labels (`alertname`, `pod`) and links for Source, Silence and Dashboard.

The links point to `localhost:3000`, so they only work on the machine with the port-forward. For links that work for everyone, set Grafana's public URL (`root_url`).

## 10. Templating: a different message per alert

The contact point is shared by every alert that uses it, so do not customize the message there. Put the text in each **alert rule** instead, using annotations. Grafana's default notification already prints a rule's Summary and Description.

In the rule, section **Configure notification message**:

- **Summary:**
  ```
  High CPU on {{ $labels.pod }}
  ```
- **Description:**
  ```
  {{ $labels.pod }} is using {{ printf "%.0f" $values.A.Value }}% of its CPU limit (alert threshold is 80%).
  ```

What the pieces mean:
- `{{ $labels.pod }}` reads a label from the alert.
- `{{ $values.A.Value }}` is the result of query A (89.0357... here); `printf "%.0f"` rounds it to 89.

Another rule can have its own text, for example a memory rule: `{{ $labels.pod }} is using {{ printf "%.0f" $values.A.Value }} MiB of memory`. You can also add your own annotations, such as a `runbook` link.

**Different channels for different alerts** is done with routing: add a label to the rule (for example `team = platform`), create a second contact point with another webhook, then in **Notification policies** add a rule "if `team = platform`, send to that contact point".

## 11. Make the alert recover (and get the Resolved message)

Grafana sends a Resolved message by default once the alert returns to Normal.

**Option A: from the UI.** Edit the `CPU Usage` rule and raise the threshold from `80` to `500`. After the next evaluation the state is Normal and a Resolved message arrives. Lower it back to `80` to fire again.

**Option B: fix the real cause.** Remove the CPU worker from the `stress` pod:

```bash
kubectl -n demo patch deploy stress --type=json \
  -p='[{"op":"replace","path":"/spec/template/spec/containers/0/args","value":["--vm","1","--vm-bytes","100M","--vm-hang","1"]}]'
```

The pod restarts with near-zero CPU. The old pod's alert ends after a couple of evaluations (2-3 minutes) and the Resolved message is sent. To bring the problem back, restore `--cpu 1` or run `kubectl -n demo rollout undo deploy/stress`. Avoid scaling `stress` to 0: with no data at all Grafana reports "No data" instead of a clean Resolved.

If the Resolved message never arrives:
- Contact point `discord-demo`, Optional Discord settings: **Disable resolved message** must be off.
- Notification policy Group interval: the default (5 minutes) can delay it; set `30s`.

## 12. Cleanup

```bash
kubectl delete -f apps.yaml
kubectl delete namespace demo
helm -n monitoring uninstall kps
kubectl delete namespace monitoring
# Helm does not remove the CRDs:
kubectl get crd -o name | grep monitoring.coreos.com | xargs kubectl delete
```

## 13. Troubleshooting

| Symptom | What to check |
|---|---|
| Pods stuck Pending in `monitoring` | `kubectl -n monitoring describe pod <pod>` (node capacity) |
| `demo` missing in the dashboard dropdown | Wait 1-2 minutes after deploying the apps |
| Test notification works, real alert does not | Alerting, Alert rules: is the state Normal, Pending, Firing or No data? |
| Threshold field not visible in the rule | Turn on **Advanced options** in the query section |
| No data in the CPU alert query | Run the query in Explore; the `stress` pod must have a CPU limit |
| Resolved message missing or late | See the end of section 11 |

## Next topics

- Route different alerts to different channels (section 10)
- Monitor an app's own metrics with a ServiceMonitor (needs the `release: kps` label with this install)
- Define alerts as YAML (PrometheusRule) and configure Alertmanager receivers through Helm values
- Add tracing (APM) with Tempo
