# Kubernetes Demo Project
Just me trying to wrap my head around kubernetes.

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