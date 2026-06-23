# Project 09 — Kubernetes & Azure Kubernetes Service (AKS)

## What You Will Learn
- What containers and Kubernetes are and why they matter
- How to create an AKS cluster via Portal and CLI
- Core Kubernetes objects: Pods, Deployments, Services, Namespaces
- How to deploy, scale, update, and roll back applications
- How to debug failing pods
- How to use the AKS Portal blades to inspect your cluster visually

---

## Background — Read This First

**Container** — A lightweight, portable package containing an app and everything it needs to run (libraries, config). Starts in seconds, uses much less memory than a VM.

**Kubernetes (K8s)** — Manages containers across multiple servers. It automatically restarts crashed containers, scales them up under load, and distributes traffic. Think of it as an OS for containers.

**AKS (Azure Kubernetes Service)** — Microsoft's managed Kubernetes. Azure manages the control plane (masters) for you. You manage your applications and worker nodes.

**Key objects:**
| Object | What It Is |
|---|---|
| **Pod** | Smallest deployable unit — one or more containers |
| **Deployment** | Manages pods — handles scaling, updates, rollbacks |
| **Service** | Stable network endpoint for pods (load balancer) |
| **Namespace** | Virtual cluster — logical isolation |
| **ConfigMap** | Non-sensitive configuration |
| **Secret** | Sensitive data (passwords, keys) |

---

## Prerequisites
- Azure subscription, CLI logged in
- `kubectl` installed: run `az aks install-cli`

---

## Step 1 — Install kubectl

### Portal (GUI)
Open the **Cloud Shell** (`>_` in the top toolbar) → Select Bash → kubectl is pre-installed in Cloud Shell.

### CLI
```bash
az aks install-cli
kubectl version --client
```

---

## Step 2 — Create an AKS Cluster

> **Cost note:** AKS charges for worker node VMs. Use a small size and delete when done.

### Portal (GUI)
1. Search **Kubernetes services** → **+ Create** → **Kubernetes cluster**
2. **Basics** tab:
   - Resource group: `rg-aks-lab` (create new)
   - Cluster name: `aks-lab-cluster`
   - Region: `UK South`
   - Kubernetes version: leave default
   - Node size: click **Choose a size** → select `Standard_B2s`
   - Node count: `1`
3. **Integrations** tab: leave defaults (managed identity is on)
4. Click **Review + create** → **Create**
5. Wait 3–5 minutes — Azure is setting up VMs, networking, and the Kubernetes control plane

**Explore the AKS Portal blades once created:**
- Click **Workloads** in the left panel → see pods, deployments, services (empty for now)
- Click **Node pools** → see your single worker node
- Click **Networking** → see the VNet and DNS settings
- Click **Insights** → see CPU/memory metrics (requires Log Analytics)

### CLI
```bash
az group create --name rg-aks-lab --location uksouth

az aks create \
  --resource-group rg-aks-lab \
  --name aks-lab-cluster \
  --node-count 1 \
  --node-vm-size Standard_B2s \
  --generate-ssh-keys \
  --enable-managed-identity
```

---

## Step 3 — Connect kubectl to the Cluster

### Portal (GUI)
1. On the AKS cluster page, click **Connect** at the top
2. The Portal shows you the exact commands to run — copy and paste them into your local terminal or Cloud Shell
3. After running them, kubectl is configured to talk to your cluster

### CLI
```bash
az aks get-credentials \
  --resource-group rg-aks-lab \
  --name aks-lab-cluster

# Verify connection
kubectl cluster-info
kubectl get nodes
kubectl get nodes -o wide
```

**What you just learned:** `az aks get-credentials` writes connection details to `~/.kube/config`. All `kubectl` commands now go to your AKS cluster.

---

## Step 4 — Namespaces

### Portal (GUI)
1. On the AKS cluster page, click **Namespaces** in the left panel (under Kubernetes resources)
2. You see: `default`, `kube-system`, `kube-public`, `kube-node-lease`

### CLI
```bash
kubectl get namespaces

# Create your own
kubectl create namespace lab-apps

kubectl get namespaces
```

**What you just learned:** `kube-system` contains Kubernetes' own components (DNS, metrics server). Never touch it unless you know what you are doing. Always put your own apps in a named namespace — not `default`.

---

## Step 5 — Run Your First Pod

### Portal (GUI)
Pods are easiest to create from YAML files. Skip ahead to Step 7 for the portal deployment creation — individual pods are rarely created via the portal.

### CLI
```bash
kubectl run nginx-pod \
  --image=nginx:latest \
  --namespace=lab-apps

# Watch it start
kubectl get pods -n lab-apps --watch
# (Ctrl+C to stop watching)

# Full details including Events (your debugging section)
kubectl describe pod nginx-pod -n lab-apps
```

**What you just learned:** The `Events` section at the bottom of `kubectl describe` is where Kubernetes tells you why a pod is in a particular state. Always read Events when a pod is not Running.

---

## Step 6 — Exec Into a Pod and View Logs

### Portal (GUI)
1. On the AKS cluster page, click **Workloads** → **Pods** tab
2. Select namespace `lab-apps`
3. Click `nginx-pod`
4. You see pod details, labels, and conditions
5. Click the **Live logs** button — see container stdout in the browser
6. To get a shell: some portal versions have a **Connect** button — or use kubectl

### CLI
```bash
# Execute a command inside the pod
kubectl exec nginx-pod -n lab-apps -- nginx -v

# Get an interactive shell
kubectl exec -it nginx-pod -n lab-apps -- /bin/bash
# Inside the pod:
ls /etc/nginx/
exit

# View logs
kubectl logs nginx-pod -n lab-apps
kubectl logs -f nginx-pod -n lab-apps   # follow live (Ctrl+C)
```

**What you just learned:** `kubectl exec` and `kubectl logs` are your two primary debugging tools inside a running container.

---

## Step 7 — Create a Deployment (Production Pattern)

A standalone pod is not production-ready. A Deployment manages pods and automatically restarts them if they crash.

### Portal (GUI)
1. On the AKS cluster page, click **Workloads** → **+ Create** → **Apply a YAML**
2. Paste the YAML below → click **Add**

### CLI
```bash
kubectl delete pod nginx-pod -n lab-apps   # delete the standalone pod first

cat > nginx-deployment.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  namespace: lab-apps
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
        ports:
        - containerPort: 80
        resources:
          requests:
            memory: "64Mi"
            cpu: "100m"
          limits:
            memory: "128Mi"
            cpu: "200m"
EOF

kubectl apply -f nginx-deployment.yaml
kubectl get pods -n lab-apps --watch
```

**What you just learned:** `replicas: 2` means Kubernetes keeps 2 running pods at all times. If one crashes, Kubernetes creates a replacement. Resource `requests` reserves capacity; `limits` caps it. Without limits, one misbehaving app can starve others on the node.

---

## Step 8 — Expose with a Service

### Portal (GUI)
1. Click **Workloads** → **Services and ingresses** → **+ Create** → **Apply a YAML**
2. Paste the service YAML and click **Add**
3. After creation, the service appears with an External IP — click it to open nginx in a browser

### CLI
```bash
cat > nginx-service.yaml << 'EOF'
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
  namespace: lab-apps
spec:
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 80
  type: LoadBalancer
EOF

kubectl apply -f nginx-service.yaml

# Watch for the external IP (takes 1-2 minutes)
kubectl get service nginx-service -n lab-apps --watch

NGINX_IP=$(kubectl get service nginx-service -n lab-apps \
  --output jsonpath='{.status.loadBalancer.ingress[0].ip}')
curl http://$NGINX_IP
```

**What you just learned:** `type: LoadBalancer` tells AKS to provision an Azure Load Balancer with a public IP. The Service's `selector: app: nginx` finds all pods with that label. Traffic is distributed across all matching pods automatically.

---

## Step 9 — Scale the Deployment

### Portal (GUI)
1. Click **Workloads** → **Deployments** → click `nginx-deployment`
2. Click **Scale** at the top → change replica count to `4` → **Scale**
3. Watch new pods appear in the Pods list

### CLI
```bash
kubectl scale deployment nginx-deployment --replicas=4 -n lab-apps
kubectl get pods -n lab-apps --watch

# Scale back down
kubectl scale deployment nginx-deployment --replicas=2 -n lab-apps
```

---

## Step 10 — Rolling Update

### Portal (GUI)
1. Click **Workloads** → **Deployments** → `nginx-deployment` → **Edit** (YAML icon)
2. Find `image: nginx:1.25` and change to `nginx:1.26` → **Save**
3. Watch the rollout in the Pods list — new pods come up before old ones are removed

### CLI
```bash
kubectl set image deployment/nginx-deployment \
  nginx=nginx:1.26 \
  --namespace=lab-apps

# Watch the rolling update
kubectl rollout status deployment/nginx-deployment -n lab-apps
```

**What you just learned:** Rolling update = new pods come up BEFORE old ones are removed. Zero downtime. Kubernetes only removes an old pod after a new one passes its health checks.

---

## Step 11 — Rollback

### Portal (GUI)
1. Click `nginx-deployment` → **Revision history** tab (if shown) or use kubectl

### CLI
```bash
kubectl rollout history deployment/nginx-deployment -n lab-apps
kubectl rollout undo deployment/nginx-deployment -n lab-apps
kubectl rollout status deployment/nginx-deployment -n lab-apps
```

---

## Step 12 — ConfigMaps and Secrets

### Portal (GUI)
1. Click **Configuration** → **Config maps** → **+ Create** → **Apply a YAML**
2. Paste the ConfigMap YAML → **Add**

### CLI
```bash
kubectl create configmap nginx-config \
  --from-literal=ENVIRONMENT=lab \
  --from-literal=LOG_LEVEL=info \
  --namespace=lab-apps

kubectl create secret generic app-secret \
  --from-literal=DB_PASSWORD=MySecretPass123 \
  --namespace=lab-apps

# View (secrets are base64 encoded)
kubectl get secret app-secret -n lab-apps -o yaml

# Decode
kubectl get secret app-secret -n lab-apps \
  --output jsonpath='{.data.DB_PASSWORD}' | base64 --decode
```

---

## Step 13 — Debugging Commands

```bash
# All resources in a namespace
kubectl get all -n lab-apps

# Describe a failing pod
kubectl describe pod <pod-name> -n lab-apps

# Logs from a crashed pod (previous instance)
kubectl logs <pod-name> -n lab-apps --previous

# Resource usage
kubectl top nodes
kubectl top pods -n lab-apps

# Find pods in a bad state
kubectl get pods -n lab-apps --field-selector=status.phase!=Running
```

---

## Step 14 — Clean Up

### Portal (GUI)
Navigate to `rg-aks-lab` → **Delete resource group** → confirm

### CLI
```bash
kubectl delete namespace lab-apps
az group delete --name rg-aks-lab --yes --no-wait
```

---

## Key Concepts Table

| Concept | What It Means |
|---|---|
| Pod | Smallest unit — one or more containers |
| Deployment | Manages pods, handles updates and rollbacks |
| Service (LoadBalancer) | Stable endpoint with Azure Load Balancer — distributes traffic |
| `kubectl describe` | Primary debug tool — always read the Events section |
| `kubectl logs` | Read container output |
| `kubectl exec -it` | Shell into a running container |
| Rolling update | New pods before old removed — zero downtime |
| Rollback | Undo to previous deployment version — emergency brake |

---

## Common Interview Questions

1. **"A pod is in CrashLoopBackOff. How do you debug it?"** — `kubectl describe pod <name>` to see Events. `kubectl logs <name> --previous` to see why it crashed last time. Common causes: app startup error, missing environment variable, out of memory, wrong image, missing config file.

2. **"What is the difference between a Pod and a Deployment?"** — A Pod is one running instance. A Deployment is a controller that manages multiple identical Pods — it ensures desired replicas are always running and handles rolling updates and rollbacks.

3. **"How does Kubernetes achieve zero-downtime deployments?"** — Rolling update strategy: new pods start with the updated image, and only after a new pod passes its readiness probe does Kubernetes terminate an old pod. `maxUnavailable` and `maxSurge` control the speed of the rollout.

---

## Next Steps
Move on to **Project 10 — Active Directory & Group Policy**.
