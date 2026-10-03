# EKS Autoscaling Demo: HPA + Cluster Autoscaler

Hands-on guide for the EKS cluster **`zidd-3`** in **`us-east-1`**.

| | HPA (Horizontal Pod Autoscaler) | Cluster Autoscaler (CA) |
|---|---|---|
| Scales | **Pods** | **Nodes** |
| Trigger | Actual CPU/memory usage above target | Pods stuck in `Pending` because nothing fits |
| Looks at | Real usage (from metrics-server) | Resource **requests** |
| Speed | Seconds to minutes | Minutes (a new EC2 instance must boot) |

**How they work together:** load increases, HPA adds pods, the nodes run out of room so new pods go `Pending`, and CA adds nodes so the pods can run. When load drops, HPA removes pods first, then CA removes the empty nodes.

---

## Prerequisites

- `aws`, `kubectl`, `eksctl`, `helm` installed and logged in to AWS
- Cluster: `zidd-3`, region: `us-east-1`, AWS account: `462096274890`
- Cluster Kubernetes version: `1.36`

Point kubectl at the cluster:

```bash
aws eks update-kubeconfig --name zidd-3 --region us-east-1
kubectl get nodes
```

---

# Part 1: HPA Demo (scale pods)

HPA watches the CPU usage of a deployment's pods and adds or removes replicas to keep average usage near a target (here 50% of the CPU request).

## 1.1 Check metrics-server

HPA reads CPU usage from **metrics-server**. Without it, HPA shows `<unknown>` and never scales.

```bash
kubectl top nodes
```

- If you see numbers, continue.
- If you see `Metrics API not available` or `ServiceUnavailable`, see [Troubleshooting](#troubleshooting-hpa-shows-unknown).

On this cluster metrics-server is an **EKS add-on**. If it is missing, install it as an add-on (not with `kubectl apply`, which conflicts with the add-on):

```bash
aws eks create-addon --cluster-name zidd-3 --region us-east-1 --addon-name metrics-server
aws eks wait addon-active --cluster-name zidd-3 --region us-east-1 --addon-name metrics-server
```

## 1.2 Deploy the demo app

The `hpa-example` image burns real CPU when it receives requests. The CPU **request** (500m) matters twice: HPA computes utilization as a percentage of it, and CA uses it to decide if pods fit on a node.

```bash
cat > hpa-demo.yaml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: php-apache
spec:
  replicas: 1
  selector:
    matchLabels:
      app: php-apache
  template:
    metadata:
      labels:
        app: php-apache
    spec:
      containers:
      - name: php-apache
        image: registry.k8s.io/hpa-example
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: "500m"
            memory: "128Mi"
          limits:
            cpu: "1"
            memory: "256Mi"
---
apiVersion: v1
kind: Service
metadata:
  name: php-apache
spec:
  selector:
    app: php-apache
  ports:
  - port: 80
EOF

kubectl apply -f hpa-demo.yaml
```

## 1.3 Create the HPA

Keep average CPU around 50% of the request, with between 1 and 20 pods.

```bash
cat > hpa.yaml <<'EOF'
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: php-apache
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: php-apache
  minReplicas: 1
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
EOF

kubectl apply -f hpa.yaml
kubectl get hpa php-apache
```

`TARGETS` may show `<unknown>/50%` for the first minute, then something like `0%/50%`.

## 1.4 Watch (use separate terminals)

```bash
# Terminal 1: HPA
kubectl get hpa php-apache -w

# Terminal 2: pods
kubectl get pods -l app=php-apache -w

# Terminal 3: nodes
kubectl get nodes -w
```

## 1.5 Generate load

```bash
kubectl run load-generator --image=busybox:1.28 --restart=Never -- /bin/sh -c "while true; do wget -q -O- http://php-apache; done"
```

For more pressure, add a second one:

```bash
kubectl run load-generator-2 --image=busybox:1.28 --restart=Never -- /bin/sh -c "while true; do wget -q -O- http://php-apache; done"
```

## 1.6 Expected result

1. HPA terminal: CPU climbs above the target, for example `250%/50%`.
2. Replicas increase step by step (1, then 4, then 8, and so on).
3. Pods are created and become `Running` while there is room on the nodes.

At this point **only HPA is working**. Pods are scaling, but nodes are not yet. Part 2 adds node scaling. If the nodes still have room, no pod goes `Pending` and no new node is needed.

## 1.7 Stop the load and watch scale-down

```bash
kubectl delete pod load-generator load-generator-2 --ignore-not-found
```

HPA waits about **5 minutes** (stabilization window) and then reduces replicas back toward 1.

---

# Part 2: Cluster Autoscaler (scale nodes)

CA watches for pods that are `Pending` because no node has enough free CPU or memory (based on requests). It then raises the desired size of the node group's Auto Scaling Group (ASG). When nodes sit mostly empty for about 10 minutes, it removes them.

## 2.1 Associate the OIDC provider

**Why:** CA runs as a pod and needs AWS permissions to change ASG sizes. IAM Roles for Service Accounts (IRSA) gives permissions to a single Kubernetes service account instead of the whole node. IRSA requires an OIDC provider linked to the cluster.

```bash
eksctl utils associate-iam-oidc-provider --cluster zidd-3 --region us-east-1 --approve
```

If it prints `already associated`, nothing more to do.

## 2.2 Node group scaling limits (console)

**Why:** CA only scales between a node group's min and max. If max equals the current node count, it can never add nodes.

1. EKS console (region **us-east-1**) → **Clusters → zidd-3 → Compute**.
2. Click the node group → **Edit**.
3. Under **Node group scaling configuration** set **Minimum**, **Maximum** (higher than current, for example 5 or 10) and **Desired** (current count).
4. Save. Repeat for every node group you want autoscaled.

## 2.3 Check the ASG tags (console)

**Why:** CA auto-discovers only the ASGs that carry two specific tags. Untagged ASGs are ignored, which is useful for node groups you want kept fixed.

1. EC2 console → **Auto Scaling → Auto Scaling Groups**.
2. Open the ASG for your node group (name looks like `eks-<nodegroup>-<uuid>`).
3. Open the **Tags** section and check for:

| Key | Value |
|---|---|
| `k8s.io/cluster-autoscaler/enabled` | `true` |
| `k8s.io/cluster-autoscaler/zidd-3` | `owned` |

4. If missing, click **Edit → Add tag**, add both, and leave "Tag new instances" off. Click **Update**.

Repeat for each ASG of the cluster. To check all node groups at once from the CLI:

```bash
aws autoscaling describe-auto-scaling-groups --region us-east-1 \
  --query "AutoScalingGroups[?contains(Tags[?Key=='eks:cluster-name'].Value, 'zidd-3')].{Name:AutoScalingGroupName,Min:MinSize,Max:MaxSize,Desired:DesiredCapacity,Tags:Tags[].Key}" \
  --output json
```

## 2.4 Create the IAM policy

**Why:** This is the list of AWS actions CA may perform: describe ASGs, change their size, and terminate instances. Scaling actions are limited to ASGs with the `enabled` tag.

```bash
cat > ca-policy.json <<'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "autoscaling:SetDesiredCapacity",
        "autoscaling:TerminateInstanceInAutoScalingGroup"
      ],
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "aws:ResourceTag/k8s.io/cluster-autoscaler/enabled": "true"
        }
      }
    },
    {
      "Effect": "Allow",
      "Action": [
        "autoscaling:DescribeAutoScalingGroups",
        "autoscaling:DescribeAutoScalingInstances",
        "autoscaling:DescribeLaunchConfigurations",
        "autoscaling:DescribeScalingActivities",
        "autoscaling:DescribeTags",
        "ec2:DescribeImages",
        "ec2:DescribeInstanceTypes",
        "ec2:DescribeLaunchTemplateVersions",
        "ec2:GetInstanceTypesFromInstanceRequirements",
        "eks:DescribeNodegroup"
      ],
      "Resource": "*"
    }
  ]
}
EOF

aws iam create-policy \
  --policy-name AmazonEKSClusterAutoscalerPolicy-zidd-3 \
  --policy-document file://ca-policy.json
```

## 2.5 Create the IAM role and service account (IRSA)

**Why:** This creates an IAM role that trusts the cluster's OIDC provider, attaches the policy above, and creates the Kubernetes service account `cluster-autoscaler` linked to that role.

```bash
eksctl create iamserviceaccount \
  --cluster zidd-3 \
  --region us-east-1 \
  --namespace kube-system \
  --name cluster-autoscaler \
  --attach-policy-arn arn:aws:iam::462096274890:policy/AmazonEKSClusterAutoscalerPolicy-zidd-3 \
  --override-existing-serviceaccounts \
  --approve
```

Verify the role annotation is on the service account:

```bash
kubectl -n kube-system get sa cluster-autoscaler -o yaml
```

You should see an `eks.amazonaws.com/role-arn` annotation.

## 2.6 Choose the Cluster Autoscaler version

CA should match the cluster's Kubernetes minor version. Check what the chart offers:

```bash
helm repo add autoscaler https://kubernetes.github.io/autoscaler
helm repo update
helm search repo autoscaler/cluster-autoscaler --versions | head -5
```

At the time of writing, the newest chart is `9.59.0` with APP VERSION `1.35.0`. The cluster is on `1.36`, so this is a one-minor-version mismatch. It usually works but is not officially tested, so watch the logs. When a 1.36 release is published, re-run the install with the new `--version` and `image.tag`.

## 2.7 Install with Helm

```bash
helm upgrade --install cluster-autoscaler autoscaler/cluster-autoscaler \
  --namespace kube-system \
  --version 9.59.0 \
  --set autoDiscovery.clusterName=zidd-3 \
  --set awsRegion=us-east-1 \
  --set rbac.serviceAccount.create=false \
  --set rbac.serviceAccount.name=cluster-autoscaler \
  --set image.tag=v1.35.0 \
  --set extraArgs.balance-similar-node-groups=true \
  --set extraArgs.skip-nodes-with-system-pods=false \
  --set extraArgs.expander=least-waste
```

What the main flags do:

- `autoDiscovery.clusterName`: find ASGs tagged for this cluster.
- `rbac.serviceAccount.create=false`: use the IRSA service account created in 2.5.
- `balance-similar-node-groups`: spread nodes evenly across similar node groups (useful with one group per AZ).
- `expander=least-waste`: when several node groups could fit the pending pods, choose the one that wastes the least capacity.

## 2.8 Verify

```bash
kubectl -n kube-system get pods -l app.kubernetes.io/name=aws-cluster-autoscaler
kubectl -n kube-system logs -f deploy/cluster-autoscaler-aws-cluster-autoscaler
```

Good: the pod is `Running`, the logs show ASGs being discovered, and there is no `AccessDenied`.

---

# Part 3: End-to-end test (HPA + CA together)

With HPA, metrics-server and CA all in place, run the full story.

## 3.1 Start watchers

```bash
# Terminal 1
kubectl get hpa php-apache -w

# Terminal 2
kubectl get pods -l app=php-apache -w

# Terminal 3
kubectl get nodes -w

# Terminal 4: autoscaler decisions
kubectl -n kube-system logs -f deploy/cluster-autoscaler-aws-cluster-autoscaler | grep -iE "scale-up|unschedulable|expanding"
```

## 3.2 Generate load

```bash
kubectl run load-generator --image=busybox:1.28 --restart=Never -- /bin/sh -c "while true; do wget -q -O- http://php-apache; done"
kubectl run load-generator-2 --image=busybox:1.28 --restart=Never -- /bin/sh -c "while true; do wget -q -O- http://php-apache; done"
```

## 3.3 What you should see

1. **HPA:** CPU goes above 50%, so replicas increase.
2. **Pods:** once the nodes are full, new pods show `Pending`.
3. **CA logs:** a scale-up decision for the node group.
4. **Nodes:** a new node joins after about 1-3 minutes.
5. **Pods:** the `Pending` pods become `Running` on the new node.

To see why a pod is pending:

```bash
kubectl describe pod -l app=php-apache | grep -A 5 Events
```

You will see `FailedScheduling` followed by `TriggeredScaleUp`.

If HPA scales but nothing goes `Pending`, the nodes have spare room. Raise the CPU request in `hpa-demo.yaml` to `"1"`, or raise `maxReplicas`, and apply again.

## 3.4 Alternative: test CA on its own (no HPA)

CA reacts to pending pods, so large **requests** are enough. No real CPU load is needed.

```bash
cat > scale-test.yaml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: scale-test
spec:
  replicas: 0
  selector:
    matchLabels:
      app: scale-test
  template:
    metadata:
      labels:
        app: scale-test
    spec:
      containers:
      - name: pause
        image: public.ecr.aws/eks-distro/kubernetes/pause:3.7
        resources:
          requests:
            cpu: "500m"
            memory: "512Mi"
EOF

kubectl apply -f scale-test.yaml
kubectl scale deployment scale-test --replicas=20
kubectl get nodes -w
```

Pick a replica count larger than your current nodes can hold.

## 3.5 Test scale-down

```bash
kubectl delete pod load-generator load-generator-2 --ignore-not-found
kubectl delete deployment scale-test --ignore-not-found
```

- HPA reduces pods after about **5 minutes**.
- After the nodes are underused, CA removes them after about **10 more minutes** (`scale-down-unneeded-time`).

Watch with `kubectl get nodes -w`. Scale-down is slow by design.

---

# Troubleshooting

## Cluster Autoscaler

| Symptom | Likely cause |
|---|---|
| `AccessDenied` in CA logs | IRSA annotation or trust policy wrong, or policy not attached |
| No ASGs discovered | Missing or wrong ASG tags, or wrong cluster name in the tag |
| Pods Pending, no scale-up | Node group already at max, or a pod request is larger than any single node |
| Log says `max node group size reached` | Raise the node group max in the EKS console |
| `ImagePullBackOff` on the CA pod | The image tag does not exist; pick a published one |
| Nodes never scale down | Pods with local storage, bare pods, restrictive PDBs, or `safe-to-evict: false` |
| Pods Pending because of taints or node selectors | Add matching `tolerations` or `nodeSelector` |

## Troubleshooting: HPA shows `<unknown>`

`cpu: <unknown>/50%` means HPA cannot read metrics. Start here:

```bash
kubectl top nodes
kubectl get apiservices | grep metrics
kubectl -n kube-system get endpointslices -l kubernetes.io/service-name=metrics-server
kubectl -n kube-system logs deploy/metrics-server --tail=30
```

Issue seen on this cluster: the APIService `v1beta1.metrics.k8s.io` showed `False (MissingEndpoints)` and the Service had no endpoints, even though the pods were `Running`. Cause: the metrics-server **Service selector** required a `k8s-app` label that the pods did not carry. Compare the two:

```bash
kubectl -n kube-system get svc metrics-server -o jsonpath='{.spec.selector}{"\n"}'
kubectl -n kube-system get pods -l app.kubernetes.io/name=metrics-server --show-labels
```

Fix: make the Service selector match the labels the pods really have.

```bash
kubectl -n kube-system patch svc metrics-server --type=json \
  -p='[{"op":"replace","path":"/spec/selector","value":{"app.kubernetes.io/name":"metrics-server","app.kubernetes.io/instance":"metrics-server"}}]'
```

If the add-on reverts the patch, reset it cleanly:

```bash
aws eks delete-addon --cluster-name zidd-3 --region us-east-1 --addon-name metrics-server
aws eks wait addon-deleted --cluster-name zidd-3 --region us-east-1 --addon-name metrics-server
aws eks create-addon --cluster-name zidd-3 --region us-east-1 --addon-name metrics-server --resolve-conflicts OVERWRITE
aws eks wait addon-active --cluster-name zidd-3 --region us-east-1 --addon-name metrics-server
```

Other common metrics-server errors:

| Log message | Fix |
|---|---|
| `x509: cannot validate certificate` | Add `--kubelet-insecure-tls` (fine for a demo) |
| `i/o timeout` on port 10250 | Node security group must allow TCP 10250 from the cluster security group |
| `no such host` | Add `--kubelet-preferred-address-types=InternalIP` |

Do **not** re-apply the upstream `components.yaml` while the EKS add-on is installed. The two conflict.

---

# Cleanup

Remove everything created by this demo. This leaves existing workloads, the metrics-server add-on and the OIDC provider untouched.

```bash
# 1. Demo workloads and HPA
kubectl delete pod load-generator load-generator-2 --ignore-not-found
kubectl delete hpa php-apache --ignore-not-found
kubectl delete deployment php-apache scale-test --ignore-not-found
kubectl delete service php-apache --ignore-not-found

# 2. Cluster Autoscaler
helm uninstall cluster-autoscaler -n kube-system

# 3. IAM role and service account
eksctl delete iamserviceaccount \
  --cluster zidd-3 \
  --region us-east-1 \
  --namespace kube-system \
  --name cluster-autoscaler

# 4. IAM policy
aws iam delete-policy --policy-arn arn:aws:iam::462096274890:policy/AmazonEKSClusterAutoscalerPolicy-zidd-3

# 5. Local files
rm -f ca-policy.json scale-test.yaml hpa-demo.yaml hpa.yaml
```

After CA is removed, extra nodes will **not** scale down by themselves. Check `kubectl get nodes` and reset the node group's min, max and desired values in the EKS console (zidd-3 → Compute → node group → Edit). Optionally remove the two `k8s.io/cluster-autoscaler/...` tags from the ASGs.

---

# Notes

- **Requests drive CA.** Pods without CPU and memory requests will not trigger scale-up correctly.
- **Protect critical pods** with `cluster-autoscaler.kubernetes.io/safe-to-evict: "false"` and PodDisruptionBudgets.
- **Multi-AZ with EBS volumes:** use one node group per AZ and keep `balance-similar-node-groups=true`.
- **Replicas:** run a single CA replica. It uses leader election, so more replicas add no capacity.
- **Alternative:** Karpenter provisions instances directly without ASGs and consolidates nodes automatically. It is worth evaluating for new clusters.
