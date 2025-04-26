---
```markdown
# 🛡️ Kubernetes Security Hardening Lab

This repository demonstrates how to harden a Kubernetes cluster using **Kyverno**, **Falco**, and **RBAC** controls. The objective of this project is to implement security best practices by:
- Enforcing security policies via **Kyverno**.
- Detecting runtime security threats with **Falco**.
- Implementing **Role-Based Access Control (RBAC)** to restrict access.

---

## 🔧 Prerequisites

Before you begin, ensure you have the following:
- A **Kubernetes cluster** (local or cloud). If you don't have one, you can use [Kind](https://kind.sigs.k8s.io/) to create a local cluster.
- **kubectl** installed and configured to interact with your cluster.
- **Helm** for installing Kyverno and Falco.
- **GitHub** account to store your configuration (optional for sharing the project).
  
**Tools Used:**
- **Kubernetes**: v1.26+
- **Kyverno**: v1.x (Policy Engine)
- **Falco**: v0.x (Runtime Security)
- **Helm**: v3.x

---

## 🚀 Setting Up Your Kubernetes Cluster

### Step 1: Create a Kubernetes Cluster (Local or Cloud)

If you're using **Kind** for a local cluster, run the following command:

```bash
kind create cluster --name security-lab
kubectl cluster-info
kubectl get nodes
```

Alternatively, use **GKE** (Google Kubernetes Engine) or **EKS** (AWS Kubernetes) for cloud-based clusters.

---

## 🔑 Set Up RBAC (Role-Based Access Control)

RBAC is essential to limit permissions based on roles. In this lab, we'll create **Admin** and **Limited User** roles.

### Step 2: Apply RBAC Policies

1. **Admin User**: Full access to the cluster

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user-binding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kube-system
```

2. **Limited User**: Access only to read Pods

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: limited-user
  namespace: default
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: default
  name: pod-reader
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods-binding
  namespace: default
subjects:
- kind: ServiceAccount
  name: limited-user
  namespace: default
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

### Step 3: Apply RBAC Policies

Run the following commands to apply the policies:

```bash
kubectl apply -f admin-user.yaml
kubectl apply -f limited-user.yaml
```

Verify access control using:

```bash
kubectl auth can-i list pods --as=system:serviceaccount:default:limited-user
```

---

## 🛡️ Install Kyverno (Policy Engine)

Kyverno is used to enforce security policies at the cluster level.

### Step 4: Install Kyverno Using Helm

Add the Kyverno Helm repository:

```bash
helm repo add kyverno https://kyverno.github.io/kyverno/
helm repo update
```

Install Kyverno:

```bash
helm install kyverno kyverno/kyverno --namespace kyverno --create-namespace
kubectl get pods -n kyverno
```

### Step 5: Apply Kyverno Policies

We'll apply three important Kyverno policies for security.

1. **Deny Privileged Containers**  
This policy will block the deployment of any privileged containers.

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: disallow-privileged
spec:
  validationFailureAction: enforce
  rules:
    - name: validate-privileged
      match:
        resources:
          kinds:
            - Pod
      validate:
        message: "Privileged mode is not allowed."
        pattern:
          spec:
            containers:
              - securityContext:
                  privileged: "false"
```

Apply the policy:

```bash
kubectl apply -f deny-privileged.yaml
```

2. **Require Specific Labels for Pods**  
This policy ensures that every pod has a specific label (e.g., `app=nginx`).

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-labels
spec:
  validationFailureAction: enforce
  rules:
    - name: validate-labels
      match:
        resources:
          kinds:
            - Pod
      validate:
        message: "Missing required label 'app'"
        pattern:
          metadata:
            labels:
              app: "?*"
```

Apply the policy:

```bash
kubectl apply -f require-labels.yaml
```

3. **Restrict Deployment of Untrusted Images**  
This policy prevents the deployment of images from untrusted registries.

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: restrict-unsafe-images
spec:
  validationFailureAction: enforce
  rules:
    - name: validate-image
      match:
        resources:
          kinds:
            - Pod
      validate:
        message: "Only trusted images allowed."
        pattern:
          spec:
            containers:
              - image: "gcr.io/trusted/*"
```

Apply the policy:

```bash
kubectl apply -f restrict-unsafe-images.yaml
```

---

## 🔒 Install Falco (Runtime Security)

Falco is used for runtime threat detection and monitoring.

### Step 6: Install Falco Using Helm

Add the Falco Helm repository:

```bash
helm repo add falcosecurity https://falcosecurity.github.io/charts
helm repo update
```

Install Falco:

```bash
helm install falco falcosecurity/falco --namespace falco --create-namespace
kubectl get pods -n falco
```

### Step 7: Check Falco Logs

```bash
kubectl logs -n falco daemonset/falco
```

---

## 💥 Simulate an Attack

Now, simulate an attack to verify if Falco is working properly.

1. **Simulate a Container Escape Attack** (exec into a pod):

```bash
kubectl run nginx --image=nginx
kubectl exec -it nginx -- bash
```

2. **Check Falco Logs for Alerts**:

```bash
kubectl logs -n falco daemonset/falco
```

If successful, you should see alerts like:

```
Notice A shell was spawned in a container with sensitive mount
```

---

## 🛠️ Final Cluster Hardening

To further secure your cluster, apply these practices:
- Disable unused Kubernetes features (e.g., hostPath volumes).
- Enable audit logs (if using GKE or EKS).
- Enforce strong Network Policies using tools like **Cilium** or **Calico**.
- Lock down API access by using the **least privilege** model for RBAC.

---

## 📂 Project Structure

```bash
k8s-security-lab/
├── README.md
├── manifests/
│   ├── rbac/
│   │   ├── admin-user.yaml
│   │   └── limited-user.yaml
│   ├── kyverno/
│   │   ├── deny-privileged.yaml
│   │   ├── require-labels.yaml
│   │   └── restrict-unsafe-images.yaml
│   ├── falco/
│   │   └── falco-values.yaml (optional)
├── attack-scenarios/
│   └── exec-into-pod.md
├── screenshots/
└── results/
    └── falco-alerts.log
```

---

## 📢 Contribute

If you find any bugs or improvements, feel free to open an issue or submit a pull request!

---

