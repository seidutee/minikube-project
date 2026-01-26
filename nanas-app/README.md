# minikube-project

# To access the app via the web, use port forwarding
NodePort is mostly conceptual

Ingress is the real production pattern

kubectl port-forward is a debugging tool only

Host + path resolution is critical for JS apps

1️⃣ Enable Ingress in Docker Desktop or install the nginx controller on the cluster

Docker Desktop comes with an NGINX ingress controller you can enable:

Open Docker Desktop → Settings → Kubernetes → Enable Ingress Controller

Apply / restart Kubernetes if prompted

Check it’s running:

kubectl get pods -n kube-system


You should see something like:

nginx-ingress-controller-xxxxx   Running

or install via kubectl
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.9.5/deploy/static/provider/cloud/deploy.yaml

Verify by running
kubectl get pods -n ingress-nginx


2️⃣ Create an Ingress resource

Let’s create a file webapp-ingress.yaml:

apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: webapp-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
    - host: webapp.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: webapp-service
                port:
                  number: 3000


webapp.local → we’ll map this to 127.0.0.1 in hosts file

service.name → must match your webapp-service

port → must match service port: 3000

3️⃣ Apply the Ingress
kubectl apply -f webapp-ingress.yaml


Verify:

kubectl get ingress


You should see:

NAME             CLASS    HOSTS         ADDRESS   PORTS   AGE
webapp-ingress   <none>   webapp.local  127.0.0.1 80     1m

4️⃣ Edit your hosts file

Add this line to your hosts file (C:\Windows\System32\drivers\etc\hosts on Windows):

127.0.0.1   webapp.local


This ensures the browser resolves webapp.local to your local Docker Desktop cluster.

5️⃣ Test in browser

Open:

http://webapp.local


The full frontend + backend should render

JS fetches /get-profile correctly

No blank page, no port-forward hacks

✅ Production-style access achieved.

6️⃣ Verify traffic in Kubernetes
kubectl describe ingress webapp-ingress
kubectl get svc -n kube-system  # make sure ingress controller is running


Everything should route through the NGINX ingress controller