# argo-hello
GitOps with Kubernetes, Helm, ArgoCD & DockerHub

This project demonstrates how to deploy a custom NGINX app with an HTML page + image into Kubernetes using Helm and manage it with ArgoCD in a GitOps workflow.

🎯 Learning Goals

Create and manage Kubernetes apps with Helm.

Use GitHub as a single source of truth (GitOps).

Manage deployments with ArgoCD.

Build & push custom Docker images.

Serve both text and images from Kubernetes.

🛠 Prerequisites

Minikube

kubectl

Helm

Docker

GitHub account

DockerHub account

🚀 Setup Instructions
1. Start Minikube
minikube start

2. Install ArgoCD
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml


Access UI:

kubectl port-forward svc/argocd-server -n argocd 8080:443


Open https://localhost:8080

Username: admin

Password:

kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d; echo

3. Clone Repo
git clone https://github.com/<your-username>/argo-hello.git
cd argo-hello

4. Create Helm Chart
helm create hello-chart

5. Build Custom Docker Image

Inside hello-chart/docker/ create:

index.html

myimage.png

Dockerfile:

FROM nginx:alpine
COPY index.html /usr/share/nginx/html/index.html
COPY myimage.png /usr/share/nginx/html/myimage.png


Build & push:

cd hello-chart/docker
docker build -t <dockerhub-username>/argo-hello:v1 .
docker push <dockerhub-username>/argo-hello:v1

6. Update Helm Chart

Edit hello-chart/values.yaml:

image:
  repository: <dockerhub-username>/argo-hello
  tag: v1
  pullPolicy: Always

service:
  type: NodePort
  port: 80

7. Clean Deployment

Replace hello-chart/templates/deployment.yaml with:

apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "hello-chart.fullname" . }}
  labels:
    {{- include "hello-chart.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      {{- include "hello-chart.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "hello-chart.selectorLabels" . | nindent 8 }}
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - name: http
              containerPort: 80
              protocol: TCP

8. ArgoCD Application

Create argo-hello-app.yaml in repo root:

apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: argo-hello
  namespace: argocd
spec:
  project: default
  source:
    repoURL: 'https://github.com/<your-username>/argo-hello.git'
    targetRevision: main
    path: hello-chart
  destination:
    server: https://kubernetes.default.svc
    namespace: demo
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true


Apply:

kubectl apply -f argo-hello-app.yaml

9. Commit & Push
git add .
git commit -m "Final GitOps setup with custom Docker image"
git push origin main

10. Verify Deployment

Check pods and service:

kubectl get pods -n demo
kubectl get svc -n demo


Open in browser:

minikube service argo-hello-hello-chart -n demo


✅ You should see your HTML message + image 🎉

🩺 Troubleshooting
App shows default NGINX page

Remove all ConfigMap mounts from Deployment.

Verify pod image:

kubectl describe deployment argo-hello-hello-chart -n demo | grep Image

New changes not appearing

Always build with a new tag:

docker build -t <dockerhub-username>/argo-hello:v2 .
docker push <dockerhub-username>/argo-hello:v2


Update values.yaml → commit → push → ArgoCD redeploys.

Service not accessible

Ensure service.type: NodePort.

Open via:

minikube service argo-hello-hello-chart -n demo

🎓 Student Exercise

Replace myimage.png with your own picture.

Build & push as v2.

Update values.yaml → commit → push.

Watch ArgoCD sync automatically.

Refresh browser → see your new image live.

✨ With this setup, you now have a full GitOps workflow:

GitHub → source of truth

ArgoCD → reconciles state

DockerHub → stores images

Kubernetes + Helm → deploys app
