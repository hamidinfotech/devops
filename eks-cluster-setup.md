# EKS Cluster Setup with EBS Storage — Reference Guide

## Prerequisites

- `eksctl` installed
- `kubectl` installed
- AWS CLI configured (`aws login --remote`)
- IAM user with the following policies attached via group:
  - `AmazonEC2FullAccess`
  - `IAMFullAccess`
  - `AWSCloudFormationFullAccess`
  - `AutoScalingFullAccess`
  - `SignInLocalDevelopmentAccess`
  - Inline policy `EksFullAccess`:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "eks:*",
      "Resource": "*"
    },
    {
      "Action": ["ssm:GetParameter", "ssm:GetParameters"],
      "Resource": [
        "arn:aws:ssm:*:<account-id>:parameter/aws/*",
        "arn:aws:ssm:*::parameter/aws/*"
      ],
      "Effect": "Allow"
    },
    {
      "Action": ["kms:CreateGrant", "kms:DescribeKey"],
      "Resource": "*",
      "Effect": "Allow"
    },
    {
      "Action": ["logs:PutRetentionPolicy"],
      "Resource": "*",
      "Effect": "Allow"
    }
  ]
}
```

---

## Step 1 — Create the Cluster

```bash
eksctl create cluster \
  --name <cluster> \
  --region eu-north-1 \
  --node-type t3.small \
  --nodes-min 3
```

> ⚠️ `t3.small` is fine for basic Kubernetes workloads but too small for the EFK stack. Use `c7i-flex.large` or `t3.medium` if running Elasticsearch.

---

## Step 2 — Install Pod Identity Agent Addon

Pod Identity is the modern way to assign IAM permissions to Kubernetes workloads. It must be installed before the EBS CSI driver.

```bash
eksctl create addon \
  --name eks-pod-identity-agent \
  --cluster <cluster> \
  --region eu-north-1
```

---

## Step 3 — Create IAM Role for EBS CSI Driver

The EBS CSI driver needs an IAM role to call AWS APIs (e.g. `CreateVolume`, `AttachVolume`) on your behalf.

```bash
# Get account ID
AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

# Create trust policy
cat > ebs-csi-trust-policy.json << EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "pods.eks.amazonaws.com"
      },
      "Action": [
        "sts:AssumeRole",
        "sts:TagSession"
      ]
    }
  ]
}
EOF

# Create the role
aws iam create-role \
  --role-name AmazonEKS_EBS_CSI_DriverRole \
  --assume-role-policy-document file://ebs-csi-trust-policy.json

# Attach the managed policy
aws iam attach-role-policy \
  --role-name AmazonEKS_EBS_CSI_DriverRole \
  --policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy
```

---

## Step 4 — Install EBS CSI Driver Addon with the Role

> ⚠️ Always pass `--service-account-role-arn` here. Without it, the driver falls back to the node instance role which lacks EBS permissions and causes `CrashLoopBackOff`.

```bash
eksctl create addon \
  --name aws-ebs-csi-driver \
  --cluster <cluster> \
  --region eu-north-1 \
  --service-account-role-arn arn:aws:iam::${AWS_ACCOUNT_ID}:role/AmazonEKS_EBS_CSI_DriverRole \
  --force
```

---

## Step 5 — Verify Addons and Pods

```bash
# Check addon status - wait until ACTIVE
eksctl get addon --cluster <cluster> --region eu-north-1

# Check CSI driver pods
# Controller should show 6/6, node pods should show 3/3
kubectl get pods -n kube-system | grep ebs
```

---

## Step 6 — Create StorageClass and PVC

Create `ebs-storage.yaml`:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: cloud-ssd
provisioner: ebs.csi.aws.com
parameters:
  type: gp2
volumeBindingMode: WaitForFirstConsumer
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mongo-pvc
spec:
  storageClassName: cloud-ssd
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 7Gi
```

```bash
kubectl apply -f ebs-storage.yaml

# PVC will show Pending - this is normal with WaitForFirstConsumer
kubectl get pvc

# PV will be empty until a pod mounts the PVC
kubectl get pv
```

> **Why Pending is normal:** `WaitForFirstConsumer` tells Kubernetes not to provision the EBS volume until a pod actually needs it. The PV is created automatically when a pod mounts the PVC.

---

## Upgrading Node Instance Type

If you need to upgrade nodes (e.g. from `t3.small` to `c7i-flex.large`):

```bash
# Step 1 - Add new node group
eksctl create nodegroup \
  --cluster <cluster> \
  --name ng-large \
  --node-type c7i-flex.large \
  --nodes-min 3 \
  --nodes-max 4 \
  --region eu-north-1

# Step 2 - Wait for new nodes to be Ready
kubectl get nodes -w

# Step 3 - Get old nodegroup name
eksctl get nodegroup --cluster <cluster> --region eu-north-1

# Step 4 - Drain old nodegroup
eksctl drain nodegroup \
  --cluster <cluster> \
  --name <old-nodegroup-name> \
  --region eu-north-1

# Step 5 - Delete old nodegroup
eksctl delete nodegroup \
  --cluster <cluster> \
  --name <old-nodegroup-name> \
  --region eu-north-1
```

---

## Free Tier Eligible Instance Types (eu-north-1)

| Instance Type | vCPU | RAM | Notes |
|---|---|---|---|
| `t3.micro` | 2 | 1GB | Too small for Kubernetes |
| `t3.small` | 2 | 2GB | Basic workloads only |
| `t4g.micro` | 2 | 1GB | ARM architecture |
| `t4g.small` | 2 | 2GB | ARM architecture |
| `c7i-flex.large` | 2 | 4GB | Recommended for EFK stack |
| `m7i-flex.large` | 2 | 8GB | Best for memory-heavy workloads |

---

## Key Notes

- PVC stays in `Pending` until a pod mounts it — expected with `WaitForFirstConsumer`
- PV is created automatically when a pod uses the PVC
- EBS volumes mount as `root:root` by default — fix with `fsGroup: 1000` in pod `securityContext` or an `initContainer` running `chown`
- Always pass `--service-account-role-arn` when creating the EBS CSI addon
- `t3.small` is too small for Elasticsearch — use `c7i-flex.large` minimum
- Delete cluster when not in use — EKS control plane costs ~$0.10/hour (~$72/month)

---

## Useful Commands

```bash
# Check nodes
kubectl get nodes

# Check all pods across all namespaces
kubectl get pods -A

# Check addon status
eksctl get addon --cluster <cluster> --region eu-north-1

# Check nodegroups
eksctl get nodegroup --cluster <cluster> --region eu-north-1

# Port forward a service
kubectl port-forward svc/<service-name> <local-port>:<service-port> -n <namespace>

# Access via SSH tunnel (when kubectl runs on EC2)
ssh -N -L <local-port>:localhost:<local-port> -i <key.pem> ubuntu@<ec2-ip>

# Force delete a stuck pod
kubectl delete pod <pod-name> -n <namespace> --force --grace-period=0

# Drain a node
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data --force

# Delete cluster when done
eksctl delete cluster --name <cluster> --region eu-north-1
```
