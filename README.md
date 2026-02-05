
# Manifest File Structure
minikube-project/
├── apps/
│   ├── bootstrap/                 # ROOT APP owns ONLY Application CRs
│   │   ├── root-app.yml            # workloads (App of Apps)
│   │   ├── nanasapp.yml             
│   │   └── teesapp.yml              
│   │
│   ├── nanas-app/                  # WORKLOADS 
│   │   ├── namespace.yml
│   │   ├── deployment.yml
│   │   ├── service.yml
│   │   ├── ingress.yml             # nanasapp.local
│   │   ├── configmap.yml
│   │   └── kustomization.yml       # optional but recommended
│   │
│   ├── tees-app/                   # REAL WORKLOADS 
│   │   ├── namespace.yml
│   │   ├── deployment.yml
│   │   ├── service.yml
│   │   ├── ingress.yml             # teesapp.local
│   │   └── kustomization.yml
│   │
│   └── _common/                    # shared manifests (optional)
│       ├── ingress-nginx/
│       └── network-policies/
│
├── environments/                   # (optional, future-ready)
│   ├── dev/
│   ├── staging/
│   └── prod/
│
└── README.md

# Docker Build and Push
# Kid 1 webapp 
docker build -t seidut/ayd:1.0 ./ayd
docker push seidut/ayd:1.0

# Kid 2 webapp 
docker build -t seidut/zay:1.1 ./zay
docker push seidut/zay:1.1


# Pre load the images to the cluster
docker pull seidut/zay:1.0 
docker pull seidut/ayd:1.0 

# Install nginx alb controller
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/cloud/deploy.yaml

# Argocd 
# Install argocd 
kubectl create namespace argocd
kubectl apply -f https://raw.githubusercontent.com/argoproj/argo-cd/v2.7.9/manifests/crds/application-crd.yaml
kubectl apply -f https://raw.githubusercontent.com/argoproj/argo-cd/v2.7.9/manifests/crds/applicationset-crd.yaml
kubectl apply -f https://raw.githubusercontent.com/argoproj/argo-cd/v2.7.9/manifests/crds/appproject-crd.yaml
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl get pods -n argocd

# Get Argocd password
kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath="{.data.password}" | base64 --decode

# Port forward to access Argocd GUI 
kubectl port-forward svc/argocd-server -n argocd 8080:80


# to delete applications that are stuck
Remove the finalizer (this is the key step)
Run:
kubectl patch application payment-app -n argocd --type=json -p='[{"op":"remove","path":"/metadata/finalizers"}]'
kubectl get applications -n argocd


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






