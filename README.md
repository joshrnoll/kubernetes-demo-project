<img src="/k8s-logo.png" height=150>

# Kubernetes Demo Project
Just me trying to wrap my head around kubernetes. Read my stream of consciousness at [NOTES.md](/NOTES.md)

![](/kubernetes-componetnts.svg)

## Usage
### Start minikube with Docker driver
```
minikube start --driver=docker
```
### Apply config
```
kubectl apply -f uptime-kuma-deployment.yml
```
### Retrieve URL with minikube service
```
minikube service uptime-kuma-service
```