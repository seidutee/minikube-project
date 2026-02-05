# Kubernetes-Argocd Project (App of Apps)
Project Overview
GitOps-Driven Microservices Deployment on Kubernetes (EKS / Minikube)
A production-style Kubernetes microservices project using GitOps with Argo CD.
Demonstrates secure service-to-service communication, ingress management, and MongoDB integration.
Focused on real-world debugging, Kustomize overlays, and operational best practices.

Steps
1. Deploy minikube or Docker Desktop Cluster
a. Run choco install minikube or 
b. Go to kubernetes on docker desktop and select kind cluster type

2.  Install argocd 
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl get pods -n argocd
kubectl port-forward svc/argocd-server -n argocd 8080:80

kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath="{.data.password}" | base64 --decode

3. Install nginx alb controller
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/cloud/deploy.yaml

4. Prepare your manifest files using the structure below
repo-root/
├── k8s/
│   ├── bootstrap/
│   │   └── root-app-dev.yaml      # App of Apps 
│   └── apps/
│       └── dev/
│           ├── web-app/
│           │   ├── deployment.yaml
│           │   ├── service.yaml
│           │   ├── ingress.yaml
│           │   
│           ├── token-app/
│           │   ├── deployment.yaml
│           │   ├── service.yaml
│           │   ├── ingress.yaml
│           │   
│           └── payment-app/
│               ├── mongo-deployment.yaml
│               ├── mongo-service.yaml
│               ├── web-deployment.yaml
│               ├── web-service.yaml
│               ├── configmap.yaml
│               ├── secret.yaml
│               └── web-ingress.yaml

| Component         | Exposure       |
| ----------------- | -------------- |
| Web App           | Ingress        |
| Token App         | Ingress        |
| Payment (MongoDB) | ClusterIP only |
| Debugging         | Port-forward   |

# to delete applications that are stuck
Remove the finalizer (this is the key step)
Run:
kubectl patch application payment-app -n argocd --type=json -p='[{"op":"remove","path":"/metadata/finalizers"}]'
kubectl get applications -n argocd
kubectl delete application payment-app -n argocd
kubectl delete application token-app -n argocd
kubectl delete application web-app -n argocd
kubectl delete application root-app -n argocd

kubectl rollout restart deployment argocd-server -n argocd
Nuclear option (rarely needed)
Only if it’s really stuck:
kubectl delete application payment-app -n argocd --force --grace-period=0
kubectl delete application token-app -n argocd --force --grace-period=0
kubectl delete application web-app -n argocd --force --grace-period=0
kubectl delete application app-of-apps -n argocd --force --grace-period=0
Final checklist (memorize this)
✔ Root app lives outside bootstrap
✔ Bootstrap contains only child Applications
✔ Root app is created once
✔ Root app never manages itself
✔ Application names never change

# Minikube is not able to pull images from dockerhub thus you have to preload your images
Web App
1. docker pull dainmusty/phone-store:latest     # listens on port 80
2. minikube image load dainmusty/phone-store:latest

Token App
1. docker pull dainmusty/effulgencetech-nodejs-img:tag          # listens on port 8080
2. minikube image load dainmusty/effulgencetech-nodejs-img:tag

Payment App (web and database)
1. docker pull mongo:5.0          
2. minikube image load mongo:5.0

3. docker pull nanajanashia/k8s-demo-app:v1.0
4. minikube image load nanajanashia/k8s-demo-app:v1.0

# Test the applications via GUI
# Application Access Options

# Option 1 — Port-forwarding (quick, not scalable)
kubectl port-forward svc/web-app -n web-app 8081:80

Pros
Fast to test
No extra resources

Cons
Manual per app
Port collisions
Not production-like
👉 Used only for short-lived debugging.

# Option 2. — Ingress-based routing (recommended)

This option mirrors how applications are exposed in production.

# Step 1 — Normalize container ports

All applications listen on the same container port (8080).

containers:
  - name: web-app
    image: dainmusty/effulgencetech-nodejs-img:tag
    ports:
      - containerPort: 8080

Repeat for:

web-app

token-app

payment-app

# Step 2 — Normalize Services

Each Service exposes port 80, forwarding to 8080 inside the Pod.

ports:
  - port: 80
    targetPort: 8080

This ensures:

Ingress always talks to port 80

Containers remain consistent

What “normalize all apps” means (in your setup)

Normalization = making all your applications look the same from Kubernetes’ point of view.

Specifically:

Every app listens on the same container port (e.g. 8080)
Kubernetes Services & Ingress don’t need to care about app-specific ports anymore.

# Why the normalisation matters
Clean Ingress rules
Reusable Helm charts
Predictable ArgoCD health checks
Fewer configuration bugs

# Local DNS Setup (Windows)
1. Open Notepad as administrator and edit your hosts file

Host file location - C:\Windows\System32\drivers\etc

Change file type from Text Documents (*.txt) → All Files to see host file

2. Open the host file and add the line below the very bottom
  127.0.0.1   apps.local 

⚠️ Make sure:

There is at least one space or tab

No # in front

No .txt extension

3. Save (Ctrl + S)

4. Flush DNS cache (important)

Open Command Prompt as Administrator and run:

ipconfig /flushdns


You should see:

Successfully flushed the DNS Resolver Cache.

✅ Verify (this MUST work)
ping apps.local


Expected:

Pinging apps.local [127.0.0.1] with 32 bytes of data
Windows ignores hosts changes unless:

File saved with admin rights

DNS cache flushed

Git Bash / WSL does not edit Windows DNS

✅ After ping works

Then your Minikube ingress URLs will work:
http://web.apps.local
http://token.apps.local
http://payment.apps.local
Next logical steps (your setup is ready)

Once DNS works, we can:

Add path-based Ingress YAML

Add health checks in ArgoCD

Prepare same layout for EKS + ALB

Do not move forward until ping apps.local works

Normalized standard (this is gold):
This confirms:
✔ Pods are running
✔ App listens on 8080
✔ Service correctly forwards 80 → 8080
Browser
 → http://token.apps.local
 → Ingress (80)
 → Service token-app (80)
 → Pod token-app (8080)
 → Node app responds ✅
Final “Normalize all apps” rule (the right way)

Normalization means:

All apps are accessed via port 80 externally,
but internally they can listen on any port they want.

External (Ingress)
web.apps.local    → 80
token.apps.local  → 80
payment.apps.local→ 80

Internal (Service → Pod)
nginx    → 80
node     → 8080
python   → 5000

Ingress never changes — Services adapt.
| App         | Container Port | Service Target | Ingress |
| ----------- | -------------- | -------------- | ------- |
| web-app     | 80             | 80             | 80      |
| token-app   | 8080           | 8080           | 80      |
| payment-app | 3000           | 3000           | 80      |


# To log into a pod and check app status and listening port
kubectl exec -it -n token-app <pod> -- sh
wget -O- http://localhost:8080


# Add HTTPS (TLS) locally with Minikube + nginx ingress, the production-correct way, without hacks.

You’ll end up with:

https://web.apps.local
https://token.apps.local
https://payment.apps.local


✔ Valid local TLS
✔ Browser trusted cert
✔ Same pattern as EKS / ALB
✔ No port-forwards needed

1️⃣ Install & setup mkcert (local CA)

mkcert creates a trusted local Certificate Authority.

Windows (PowerShell – Admin)
choco install mkcert


OR download from:
https://github.com/FiloSottile/mkcert/releases

Initialize local CA
mkcert -install


This adds a trusted root cert to your OS & browser.

2️⃣ Generate certs for your apps

From any directory (repo root is fine):

mkcert \
  web.apps.local \
  token.apps.local \
  payment.apps.local


This creates two files:

web.apps.local+2.pem
web.apps.local+2-key.pem


(rename is fine, content matters)

3️⃣ Create a Kubernetes TLS Secret (once)

We’ll create one shared TLS secret.

kubectl create secret tls apps-local-tls \
  --cert=web.apps.local+2.pem \
  --key=web.apps.local+2-key.pem \
  -n ingress-nginx


✅ Stored where ingress controller runs
✅ Reusable by all ingresses

Verify:

kubectl get secret -n ingress-nginx

4️⃣ Update Ingress to use HTTPS
Example: token-app ingress (final)
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: token-app
  namespace: token-app
spec:
  ingressClassName: nginx

  tls:
    - hosts:
        - token.apps.local
      secretName: apps-local-tls

  rules:
    - host: token.apps.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: token-app
                port:
                  number: 80


Do the same TLS block for:

web-app

payment-app

Only the host + service name change.

5️⃣ Restart ingress (important)
kubectl rollout restart deployment ingress-nginx-controller -n ingress-nginx

6️⃣ Test 🎉

Open browser:

https://token.apps.local
https://web.apps.local


✅ Green lock
✅ No browser warnings
✅ Real TLS

🧠 Key production lessons you just implemented
Local CA	mkcert
TLS secrets	Kubernetes-native
Ingress TLS	spec.tls
No hacks	No port-forward
Cloud parity	Matches ALB

This exact pattern maps directly to:

AWS ALB + ACM

GKE Managed Certs

AKS App Gateway

Only the cert source changes.

# NetworkPolicies are where Kubernetes becomes “real security” 🔐
Default Kubernetes behavior (bad):

Every pod can talk to every pod in every namespace

Zero-trust model (good):

Nothing talks to anything unless explicitly allowed

Default deny per namespace

Allow ingress traffic only from ingress-nginx

Allow DNS (required!)

(Optional) Allow app-to-app communication later

1️⃣ Prerequisite check (important)

NetworkPolicies only work if your CNI supports them.

Minikube drivers that support NetworkPolicy:

✅ docker

✅ containerd

❌ none (old VM drivers)

Verify your CNI:

kubectl get pods -n kube-system | grep -E "calico|cilium|weave"


Minikube usually uses calico, so you’re good.

2️⃣ Default-deny policy (per app namespace)

This is the foundation of zero-trust.

📄 networkpolicy-default-deny.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: token-app
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress

What this does

❌ Blocks ALL inbound traffic

❌ Blocks ALL outbound traffic

Applies to every pod in token-app

⚠️ If you stop here → app breaks (expected)

3️⃣ Allow ingress traffic from nginx ingress controller

Your apps must accept traffic only from ingress-nginx.

📄 networkpolicy-allow-ingress-nginx.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-ingress-from-nginx
  namespace: token-app
spec:
  podSelector:
    matchLabels:
      app: token-app

  policyTypes:
    - Ingress

  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: ingress-nginx
      ports:
        - protocol: TCP
          port: 8080

Why this is secure

✔ Only ingress controller can reach the app
✔ No other namespace can talk to it
✔ No pod-to-pod lateral movement

4️⃣ Allow DNS (mandatory or everything breaks)

Every pod needs DNS to function.

📄 networkpolicy-allow-dns.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
  namespace: token-app
spec:
  podSelector: {}
  policyTypes:
    - Egress
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
      ports:
        - protocol: UDP
          port: 53
        - protocol: TCP
          port: 53


Without this:

❌ Images won’t pull

❌ Services won’t resolve

❌ HTTP clients fail

5️⃣ Apply same pattern to ALL apps

Repeat these 3 policies for:

web-app

payment-app

Only change:

namespace

app: label

target port (80 vs 8080)

Example port mapping
App	NetworkPolicy port
web-app	80
token-app	8080
payment-app	8080
6️⃣ Verify zero-trust is working
🔍 Test 1 — ingress still works
https://token.apps.local

🔍 Test 2 — block lateral traffic
kubectl exec -it -n web-app <pod> -- wget -O- http://token-app.token-app.svc.cluster.local


❌ Should FAIL
✔ Zero-trust confirmed

🔍 Test 3 — ArgoCD health

ArgoCD should remain Healthy because:

It reads Kubernetes state

It does NOT need network access to pods

7️⃣ Folder structure (recommended)

Add this to each app:

apps/dev/token-app/
├── deployment.yaml
├── service.yaml
├── ingress.yaml
└── networkpolicies/
    ├── default-deny.yaml
    ├── allow-ingress-nginx.yaml
    └── allow-dns.yaml


ArgoCD will apply them automatically.

🔐 What security you just implemented (real-world)

✔ Namespace isolation
✔ Ingress-only exposure
✔ No lateral movement
✔ DNS-only egress
✔ Cloud-grade zero-trust

This is exactly what security teams demand in production EKS clusters.

# Installation of Prometheus and Grafana

# Grafana UI
kubectl port-forward -n monitoring svc/monitoring-grafana 3000:80
http://localhost:3000

# Password
kubectl get secret -n monitoring monitoring-grafana -o jsonpath="{.data.admin-password}" | base64 --decode

# Add prometheus as a datasource
url required - http://monitoring-kube-prometheus-prometheus.monitoring.svc:9090


# Prometheus
kubectl port-forward -n monitoring svc/monitoring-kube-prometheus-prometheus 9090:9090
http://localhost:9090

How to Verify the Service Name (Optional)
kubectl get svc -n monitoring



3. Verify Alertmanager (for TestAlert)
kubectl port-forward -n monitoring svc/monitoring-kube-prometheus-alertmanager 9093:9093
# Access locally via http://localhost:9093



# Persistence (PVC)

Ensure data survives pod restart
