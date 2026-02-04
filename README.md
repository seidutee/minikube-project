# Argocd 
# Install argocd 
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl get pods -n argocd

# Port forward to access Argocd GUI 
kubectl port-forward svc/argocd-server -n argocd 8080:80

# Get Argocd password
kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath="{.data.password}" | base64 --decode



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





