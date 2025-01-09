<img src="/k8s-logo.png" height=150>

# Kubernetes Demo Project
Just me trying to wrap my head around kubernetes. 

This was all done, initially, on a local [minikube](https://minikube.sigs.k8s.io/docs/start/?arch=%2Flinux%2Fx86-64%2Fstable%2Fbinary+download) cluster. Later I spun up a cluster on Proxmox using [Talos linux](https://talos.dev).

Read my stream of consciousness at [NOTES.md](/NOTES.md). The below diagram are some notes I drew up while watching [this](https://www.youtube.com/watch?v=X48VuDVv0do&t=6409s) youtube video on Kubernetes.

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
## Moving to Talos
At a certain point, I moved away from using minikube and spun up a cluster on Proxmox using [Talos linux](https://talos.dev). Read those notes [here](/TALOS.md).