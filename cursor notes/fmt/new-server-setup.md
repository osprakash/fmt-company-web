# FlowMind Server Setup Guide

> **Target**: Scaleway VM (16GB RAM)  
> **Stack**: K3s + ArgoCD + Kong

---

## 📦 1. Install K3s (Lightweight Kubernetes)

```bash
# Install K3s
curl -sfL https://get.k3s.io | sh -s - --write-kubeconfig-mode 644

# Verify installation
kubectl get nodes
```

---

## 🔄 2. ArgoCD Installation

### 2.1 Create namespace and install

```bash
# Create namespace
kubectl create namespace argocd

# Install ArgoCD
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Wait for pods to be ready
kubectl wait --for=condition=ready pod -l app.kubernetes.io/name=argocd-server -n argocd --timeout=300s
```

### 2.2 Get admin password

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d && echo
```

### 2.3 Install ArgoCD CLI

```bash
curl -sSL -o argocd-linux-amd64 https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
chmod +x argocd-linux-amd64
sudo mv argocd-linux-amd64 /usr/local/bin/argocd
```

### 2.4 Expose ArgoCD UI via NodePort

```bash
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "NodePort", "ports": [{"port": 443, "targetPort": 8080, "nodePort": 30443}]}}'
```

### 2.5 Access ArgoCD

- **URL**: `https://<your-scaleway-vm-ip>:30443`
- **Username**: `admin`
- **Password**: (from step 2.2)

---

## 🌐 3. Kong API Gateway Installation

### 3.1 Set kubeconfig

```bash
# Set kubeconfig environment variable
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml

# Verify it works
kubectl get nodes
```

### 3.2 Disable Traefik (recommended to avoid conflicts)

Check if Traefik is running:

```bash
kubectl get pods -n kube-system | grep traefik
```

If Traefik is running, disable it:

```bash
# Stop K3s
sudo systemctl stop k3s

# Edit K3s config
sudo nano /etc/rancher/k3s/config.yaml
```

Add this to the file:

```yaml
disable:
  - traefik
```

Restart K3s:

```bash
sudo systemctl start k3s
```

### 3.3 Install Helm

```bash
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod +x get_helm.sh
./get_helm.sh
```

### 3.4 Install Kong

```bash
# Ensure kubeconfig is set
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml

# Add Kong Helm repository
helm repo add kong https://charts.konghq.com
helm repo update

# Install Kong
helm install kong kong/ingress -n kong --create-namespace --set proxy.type=LoadBalancer
```

### 3.5 Verify Kong installation

```bash
# Check pods
kubectl get pods -n kong

# Check services
kubectl get svc -n kong
```

---

## ✅ Quick Reference

| Component | Access | Port |
|-----------|--------|------|
| K8s API | `https://<VM-IP>:6443` | 6443 |
| ArgoCD UI | `https://<VM-IP>:30443` | 30443 |
| Kong Proxy | `http://<VM-IP>` | 80/443 |

---

## 🔧 Troubleshooting

### Check pod status
```bash
kubectl get pods -A
```

### View pod logs
```bash
kubectl logs -f <pod-name> -n <namespace>
```

### Describe a pod for errors
```bash
kubectl describe pod <pod-name> -n <namespace>
```

---

## 🚀 4. Deploy FlowMind Applications

### 4.1 Add GitHub repo to ArgoCD

```bash
# Create the repository secret
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: fmt-gitops-repo
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: repository
stringData:
  type: git
  url: https://github.com/osprakash/fmt-gitops.git
  username: osprakash
  password: <YOUR_GITHUB_PAT>
EOF
```

### 4.2 Create the dev namespace and registry secret

```bash
# Create dev namespace
kubectl create namespace dev

# Create Scaleway registry secret
kubectl create secret docker-registry scw-registry-secret \
  --docker-server=rg.fr-par.scw.cloud \
  --docker-username=nologin \
  --docker-password=<YOUR_SCW_SECRET_KEY> \
  -n dev
```

### 4.3 Deploy your applications

```bash
# Deploy UI
kubectl apply -f https://raw.githubusercontent.com/osprakash/fmt-gitops/main/src/apps/fmt-core-ui/dev/application.yaml

# Deploy API
kubectl apply -f https://raw.githubusercontent.com/osprakash/fmt-gitops/main/src/apps/fmt-core-services/dev/application.yaml
```

### 4.4 Check status

```bash
# Check ArgoCD applications
kubectl get applications -n argocd

# Check pods in dev namespace
kubectl get pods -n dev

# Check ingress
kubectl get ingress -n dev
```

### 4.5 Access ArgoCD UI

```bash
# Get the NodePort
kubectl get svc argocd-server -n argocd

# Get password
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d && echo
```

Access ArgoCD at: `http://<VM_IP>:30443`



TODO : postgres db setup
