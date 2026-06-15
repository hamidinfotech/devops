# Alertmanager + Slack Integration on EKS
> Reference guide — kube-prometheus-stack (Helm release: `kps`, namespace: `default`)

---

## Concepts

### How the alert pipeline works

```
Prometheus  →  evaluates PrometheusRule CRDs  →  fires alerts
     ↓
Alertmanager  →  groups / deduplicates  →  routes to receivers (Slack, email, etc.)
```

### Key CRDs installed by kube-prometheus-stack

| CRD | Purpose |
|-----|---------|
| `prometheuses.monitoring.coreos.com` | Declares a Prometheus server (operator creates the StatefulSet) |
| `alertmanagers.monitoring.coreos.com` | Declares an Alertmanager deployment |
| `prometheusrules.monitoring.coreos.com` | Alert and recording rules loaded by Prometheus |
| `servicemonitors.monitoring.coreos.com` | Tells Prometheus which services to scrape |

`kubectl get prometheus` lists custom resources, **not** a built-in Kubernetes Service type.

---

## What is a CRD (Custom Resource Definition)?

Kubernetes ships with built-in resource types: Pod, Deployment, Service, ConfigMap, etc. A **CRD** lets anyone extend the Kubernetes API with entirely new resource types — without modifying Kubernetes itself.

### How it works

```
CRD registration  →  new API endpoint appears in Kubernetes
     ↓
You create objects of that type (Custom Resources)
     ↓
An Operator (controller) watches those objects and acts on them
```

Think of it in two parts:

| Part | Analogy | Example |
|------|---------|---------|
| **CRD** | The class definition / schema | "A `Prometheus` object has fields: version, replicas, storage..." |
| **Custom Resource** | An instance of that class | `kubectl get prometheus -n default` — the actual object you created |
| **Operator** | The controller that reacts | Prometheus Operator watches `Prometheus` objects and creates StatefulSets, config, etc. |

### The operator pattern

The operator pattern is what makes CRDs useful. An operator is just a controller loop running in the cluster:

```
Watch CRD objects  →  compare desired state vs actual state  →  reconcile
```

This is the same pattern Kubernetes uses internally (Deployment controller → ReplicaSet → Pods), except CRDs let third parties plug into it. When you install kube-prometheus-stack, the Prometheus Operator registers its CRDs and starts watching them. When you apply a `PrometheusRule`, the operator detects it and reloads Prometheus config — you never touch Prometheus directly.

### Useful CRD commands

```bash
# List all CRDs in the cluster
kubectl get crd

# Filter to monitoring-related ones
kubectl get crd | grep monitoring.coreos.com

# Inspect the schema of a CRD
kubectl describe crd prometheusrules.monitoring.coreos.com

# List all custom resources of a type
kubectl get prometheusrules -n default
kubectl get servicemonitors -n default
kubectl get alertmanagers -n default
```

---

## Helm & Release Info

```bash
# List all releases across all namespaces
helm list -A

# See only what YOU have overridden (your effective values.yaml)
helm get values kps -n default

# Check the ruleSelector — PrometheusRule labels must match this
kubectl get prometheus -n default -o jsonpath='{.items[0].spec.ruleSelector}'
```

---

## Add / Update Helm Repo

Helm repos are stored locally on your machine — they are NOT saved with the release in the cluster.
You must re-add them on any new machine or shell.

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# Verify
helm repo list
```

---

## Alertmanager values.yaml

Create `alertmanager-values.yaml` with only the fields you want to override:

```yaml
alertmanager:
  config:
    global:
      resolve_timeout: 5m
      slack_api_url: 'https://hooks.slack.com/services/XXX/YYY/ZZZ'

    route:
      receiver: 'slack-notifications'
      group_by: ['alertname', 'namespace']
      group_wait: 30s       # wait before sending first notification
      group_interval: 5m    # wait before sending updates for a group
      repeat_interval: 4h   # re-send if still firing
      routes:
        - receiver: 'null'
          matchers:
            - alertname = "Watchdog"   # always-firing heartbeat — silence it

    receivers:
      - name: 'null'
      - name: 'slack-notifications'
        slack_configs:
          - channel: '#your-channel-name'
            send_resolved: true
            title: '{{ if eq .Status "firing" }}🔥{{ else }}✅{{ end }} [{{ .Status | toUpper }}] {{ .CommonLabels.alertname }}'
            text: >-
              {{ range .Alerts }}
              *Alert:* {{ .Labels.alertname }} ({{ .Labels.severity }})
              *Summary:* {{ .Annotations.summary }}
              *Description:* {{ .Annotations.description }}
              *Namespace:* {{ .Labels.namespace }}
              {{ end }}
```

---

## Install & Upgrade Commands

```bash
# First-time install (idiomatic, with namespace creation)
helm upgrade --install kps prometheus-community/kube-prometheus-stack \
  --version 86.2.2 \
  -n monitoring \
  --create-namespace \
  -f values.yaml

# Upgrade existing release (pin version to avoid unintended chart upgrades)
helm upgrade kps prometheus-community/kube-prometheus-stack \
  --version 86.2.2 \
  -n default \
  --reuse-values \
  -f alertmanager-values.yaml
```

> `upgrade --install` is idempotent — installs if absent, upgrades if present. Prefer it in scripts/CI.

---

## Port Forwarding

There are two different tools for port forwarding, used at different layers. Understanding both is important on EKS since your cluster runs remotely.

### kubectl port-forward — forward a port from inside the cluster to your local machine

`kubectl port-forward` opens a tunnel through the Kubernetes API server. Traffic goes: `your browser → localhost → kubectl → k8s API → pod/service`. The target service never needs to be exposed publicly.

```bash
# Syntax
kubectl port-forward -n <namespace> <resource> <local-port>:<remote-port>

# Alertmanager UI  →  http://localhost:9093
kubectl port-forward -n default svc/kps-alertmanager 9093

# Prometheus UI  →  http://localhost:9090
kubectl port-forward -n default svc/kps-prometheus 9090

# Grafana UI  →  http://localhost:3000
kubectl port-forward -n default svc/kps-grafana 3000

# Kibana (if running EFK stack)  →  http://localhost:5601
kubectl port-forward -n default svc/kibana 5601

# Run in background (add & at the end)
kubectl port-forward -n default svc/kps-alertmanager 9093 &

# Kill all background port-forwards
pkill -f "kubectl port-forward"
```

> **Tip:** port-forward stays alive only while the command runs. If the pod restarts, the tunnel drops and you need to re-run it. For persistent access, use a LoadBalancer or Ingress instead.

---

### ssh -L — forward a port from a remote machine to your local machine

`ssh -L` is used when `kubectl` is running on a **remote machine** (e.g., an EC2 bastion/jump host) and you want to reach its localhost ports from your own laptop. Traffic goes: `your browser → localhost → SSH tunnel → remote machine → pod`.

```bash
# Syntax
ssh -L <local-port>:<remote-host>:<remote-port> <user>@<ssh-host>

# Most common pattern: tunnel to localhost on the remote machine
ssh -L 9093:localhost:9093 ec2-user@<bastion-ip>

# With an identity file (common for EC2)
ssh -L 9093:localhost:9093 -i ~/.ssh/my-key.pem ec2-user@<bastion-ip>

# Tunnel multiple ports in one SSH connection
ssh -L 9093:localhost:9093 \
    -L 9090:localhost:9090 \
    -L 3000:localhost:3000 \
    -i ~/.ssh/my-key.pem ec2-user@<bastion-ip>

# -N flag: open tunnel only, no interactive shell
ssh -N -L 9093:localhost:9093 -i ~/.ssh/my-key.pem ec2-user@<bastion-ip>
```

---

### Combining both — the typical EKS workflow

When your kubeconfig points to EKS from your local machine, you usually only need `kubectl port-forward` directly:

```
your laptop  →  kubectl port-forward (via kubeconfig/EKS API)  →  pod
```

But if you're working through a bastion host (EC2 instance with kubectl installed on it):

```
your laptop  →  ssh -L  →  bastion EC2  →  kubectl port-forward  →  pod
```

Step by step:

```bash
# Step 1: on the bastion, start kubectl port-forward
kubectl port-forward -n default svc/kps-alertmanager 9093 &

# Step 2: on your laptop, open SSH tunnel to the bastion
ssh -N -L 9093:localhost:9093 -i ~/.ssh/my-key.pem ec2-user@<bastion-ip>

# Step 3: open http://localhost:9093 in your local browser
```

---

### Comparison

| | `kubectl port-forward` | `ssh -L` |
|---|---|---|
| **What it tunnels** | Cluster service/pod → your machine | Remote machine port → your machine |
| **Requires** | kubeconfig access to cluster | SSH access to remote host |
| **Typical use** | Local kubectl → EKS pod | Bastion host → your laptop |
| **Stays alive** | Until pod restarts or Ctrl+C | Until SSH session ends |

---

## Verify the Config Loaded

Port-forward Alertmanager and Prometheus (see Port Forwarding section for full reference):

```bash
kubectl port-forward -n default svc/kps-alertmanager 9093
kubectl port-forward -n default svc/kps-prometheus 9090
```

- Alertmanager UI → **Status** tab: check route tree and receivers
- Prometheus UI → **Alerts** tab: check your rules appear and their state

---

## Test Alertmanager → Slack (Manual Alert)

Fire a fake alert directly at the Alertmanager API (while port-forwarding on 9093):

```bash
curl -XPOST http://localhost:9093/api/v2/alerts \
  -H "Content-Type: application/json" \
  -d '[
    {
      "labels": {
        "alertname": "TestSlackAlert",
        "severity": "warning",
        "namespace": "default"
      },
      "annotations": {
        "summary": "Testing Slack integration",
        "description": "If you see this in Slack, the pipeline works."
      }
    }
  ]'
```

You should receive a Slack message within ~30 seconds (`group_wait`).

---

## Custom PrometheusRule Example

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: custom-alerts
  namespace: default
  labels:
    release: kps   # ⚠️ must match your Helm release name — operator won't pick it up without this
spec:
  groups:
    - name: custom.rules
      rules:
        - alert: PodRestartingTooOften
          expr: increase(kube_pod_container_status_restarts_total[15m]) > 3
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "Pod {{ $labels.pod }} is restarting frequently"
            description: "Pod {{ $labels.pod }} in {{ $labels.namespace }} restarted more than 3 times in 15 minutes."
```

```bash
kubectl apply -f rules/custom-alerts.yaml

# Trigger it with a crash-looping pod
kubectl run crashtest --image=busybox -- /bin/sh -c "exit 1"

# Clean up
kubectl delete pod crashtest
```

---

## Recommended Repo Layout

```
infra/
├── monitoring/
│   ├── values.yaml            # kube-prometheus-stack overrides (no secrets!)
│   ├── rules/
│   │   └── custom-alerts.yaml # PrometheusRule manifests
│   └── README.md              # exact install command + version pinned
└── <cluster-name>/
    └── ...
```

---

## Keeping the Webhook Secret Out of Git

| Option | How |
|--------|-----|
| **Env var injection** | Pass `--set alertmanager.config.global.slack_api_url=$SLACK_WEBHOOK` at install time; put placeholder in values.yaml |
| **AlertmanagerConfig CRD + k8s Secret** | Store webhook in a Secret, reference from `AlertmanagerConfig`; operator merges it |
| **SOPS / helm-secrets** | Encrypt values.yaml so it can be committed safely |
| **External Secrets + AWS Secrets Manager** | Secret lives in AWS, syncs into the cluster (best for EKS production) |

---

## Alertmanager HA (Cluster Status)

`Status: disabled` in the Alertmanager UI is **normal** with 1 replica. It refers to Alertmanager's own gossip-based HA clustering, not your Kubernetes cluster.

To enable (optional, costs memory):

```yaml
alertmanager:
  alertmanagerSpec:
    replicas: 2
```

With 2+ replicas: Prometheus sends every alert to all replicas; gossip protocol ensures only one sends the Slack message (no duplicates). Add pod anti-affinity in production to spread replicas across nodes.

---

## Quick Diagnostic Commands

```bash
# Check all monitoring pods
kubectl get pods -n default | grep kps

# Check all monitoring services
kubectl get svc -n default | grep kps

# Check PrometheusRule objects loaded
kubectl get prometheusrules -n default

# See Alertmanager config secret (what the operator actually generated)
kubectl get secret alertmanager-kps-alertmanager -n default -o jsonpath='{.data.alertmanager\.yaml}' | base64 -d

# Alertmanager logs
kubectl logs -n default -l app.kubernetes.io/name=alertmanager --tail=50

# Operator logs (if rules aren't being picked up)
kubectl logs -n default -l app.kubernetes.io/name=prometheus-operator --tail=50

# Trace the object chain
kubectl get prometheus -n default          # custom resource
kubectl get statefulset -n default         # what operator created from it
kubectl get pods -n default                # running Prometheus pods
```
