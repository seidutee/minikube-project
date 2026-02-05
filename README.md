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

# Docker
# Kid webapp 1
docker build -t seidut/ayd:1.0 ./ayd
docker push seidut/ayd:1.0

# Kid webapp 2
docker build -t seidut/zay:1.1 ./zay
docker push seidut/zay:1.1


# Pre load the images to the cluster
docker pull seidut/zay:1.0 
docker pull seidut/ayd:1.0 


# Install Prometheus and Grafana
kubectl apply





