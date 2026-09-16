# Argo CD GitOps Deployment on AWS EKS

**Objective:** Create an EKS cluster, install Argo CD, connect a GitHub repository, deploy an Nginx application using GitOps, and clean up the environment.

### Architecture

```text
Developer
   |
   | git push
   v
GitHub Repository
atulkamble/argocd-app
   |
   | monitored by
   v
+-------------------+
|      Argo CD      |
|   Namespace:      |
|      argocd       |
+---------+---------+
          |
          | Sync
          v
+-------------------+
|    AWS EKS        |
|   mycluster       |
|                   |
| Deployment: nginx |
| Service: nginx    |
+-------------------+
```

### 1. Create the EKS Cluster

```bash
eksctl create cluster \
  --name mycluster \
  --region us-east-1 \
  --nodegroup-name mynodes \
  --node-type t3.medium \
  --nodes 2 \
  --nodes-min 2 \
  --nodes-max 2 \
  --managed
```

This creates an EKS cluster named `mycluster` with **2 managed EC2 worker nodes**.

### 2. Update kubeconfig

```bash
aws eks update-kubeconfig \
  --name mycluster \
  --region us-east-1
```

Verify the current Kubernetes context:

```bash
kubectl config current-context
```

### 3. Check kubectl

```bash
kubectl version --client
```

### 4. Verify the EKS Cluster

```bash
kubectl get nodes
kubectl get ns
kubectl get svc
```

Expected node status:

```text
NAME                         STATUS   ROLES    AGE
ip-xxx.ec2.internal          Ready    <none>   ...
ip-xxx.ec2.internal          Ready    <none>   ...
```

### 5. Clone the GitHub Repository

Repository:

[atulkamble/argocd-app](https://github.com/atulkamble/argocd-app?utm_source=chatgpt.com)

Clone it:

```bash
git clone https://github.com/atulkamble/argocd-app
cd argocd-app
code .
```

The repository should contain Kubernetes manifests such as:

```text
argocd-app/
├── deployment.yaml
└── service.yaml
```

Make sure the manifests either specify a namespace or are deployed through Argo CD to the intended destination namespace.

### 6. Install Argo CD

Create the namespace:

```bash
kubectl create namespace argocd
```

Verify:

```bash
kubectl get ns
```

Install Argo CD:

```bash
kubectl apply \
  -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

If you specifically encounter server-side apply conflicts, you can use:

```bash
kubectl apply \
  --server-side \
  --force-conflicts \
  -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Check Argo CD pods:

```bash
kubectl get pods -n argocd
```

Wait until the required pods are `Running`.

### 7. Access Argo CD Using Port Forwarding

Run in the foreground:

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Keep this terminal open.

Alternatively, run it in the background:

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443 \
  > /tmp/argocd-port-forward.log 2>&1 &
```

On macOS, to keep it running after the terminal session ends:

```bash
nohup kubectl port-forward svc/argocd-server -n argocd 8080:443 \
  > /tmp/argocd-port-forward.log 2>&1 &
```

Check the process:

```bash
ps aux | grep "kubectl port-forward"
```

Check the log:

```bash
cat /tmp/argocd-port-forward.log
```

### 8. Retrieve the Argo CD Admin Password

Username:

```text
admin
```

Retrieve the generated password:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d
echo
```

**Important:** Use the password returned by this command. Do not hard-code or share the generated admin password in lab documentation.

### 9. Open the Argo CD UI

Open:

```text
https://localhost:8080
```

Login with:

```text
Username: admin
Password: <password-from-step-8>
```

A browser certificate warning can occur because the local port-forward uses Argo CD's HTTPS endpoint.

### 10. Create the Argo CD Application

In the Argo CD UI, click:

```text
+ NEW APP
```

Configure:

```text
Application Name : nginx-app
Project          : default

Repository URL   : https://github.com/atulkamble/argocd-app
Revision         : main
Path             : .

Cluster URL      : https://kubernetes.default.svc
Namespace        : default
```

For a basic first lab, you can leave sync manual.

### 11. Create and Sync

Click:

```text
CREATE
```

Then open `nginx-app` and select:

```text
SYNC
```

Argo CD follows this flow:

```text
GitHub
   |
   v
Argo CD detects manifests
   |
   v
Compare Git vs Cluster
   |
   v
SYNC
   |
   v
Kubernetes Deployment + Service
   |
   v
Nginx Pods
```

### 12. Verify the Deployment

```bash
kubectl get deployments
kubectl get pods
kubectl get svc
```

For more details:

```bash
kubectl get all
```

You can also check the Argo CD application resources from the UI.

### GitOps Test

Modify your Kubernetes manifest, for example:

```yaml
spec:
  replicas: 3
```

Commit and push:

```bash
git add .
git commit -m "Scale nginx to 3 replicas"
git push origin main
```

Argo CD will detect that the Git repository and cluster are different.

```text
Git change
    |
    v
Argo CD
    |
    | OutOfSync
    v
SYNC
    |
    v
EKS updated
```

After syncing:

```bash
kubectl get pods
```

You should see the updated desired state reflected in EKS.

### 13. Stop Port Forwarding

Find it:

```bash
ps aux | grep "kubectl port-forward"
```

Stop the Argo CD port-forward process:

```bash
pkill -f "kubectl port-forward svc/argocd-server"
```

Verify:

```bash
ps aux | grep "kubectl port-forward"
```

### 14. Delete the EKS Cluster

When the lab is finished:

```bash
eksctl delete cluster \
  --name mycluster \
  --region us-east-1
```

This step is important because the EKS cluster and associated AWS resources can continue generating charges.

## Key Points to Remember

* **Git is the source of truth** in GitOps.
* **Argo CD continuously compares** the desired state in Git with the Kubernetes cluster.
* `OutOfSync` means Git and the live cluster differ.
* `Synced` means the live state matches Git.
* Argo CD is installed in the `argocd` namespace.
* The Nginx application in this lab is deployed to the `default` namespace.
* `https://kubernetes.default.svc` represents the Kubernetes cluster where Argo CD itself is running.
* Port forwarding is suitable for **lab/local access**, not a typical production exposure method.
* Retrieve the initial admin password dynamically instead of storing it in documentation.
* Always delete temporary EKS clusters after practice to avoid unnecessary AWS charges.
