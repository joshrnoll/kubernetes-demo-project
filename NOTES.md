### First Deployment
- Minikube running with QEMU driver on local machine
- used the following deployment and service YAML:
```YAML
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: uptime-kuma
  labels:
    app: uptime-kuma
spec:
  replicas: 1
  selector:
    matchLabels:
      app: uptime-kuma
  template:
    metadata:
      labels:
        app: uptime-kuma
    spec:
      containers:
      - name: uptime-kuma
        image: louislam/uptime-kuma
        ports:
        - containerPort: 3001
---
apiVersion: v1
kind: Service
metadata:
  name: uptime-kuma-service
spec:
  selector:
    app: uptime-kuma
  type: LoadBalancer  
  ports:
    - protocol: TCP
      port: 3001
      targetPort: 3001
      nodePort: 30000
```
- kubectl apply -f uptime-kuma-deployment.yml
- Got "MK_UNIMPLEMENTED" error when running ```minikube service uptime-kuma-service``` to get URL. Google showed that minikube service was unavailable when using QEMU. Saw an option to use Docker instead.
- ran ```minikube delete``` and redeployed minikube with ```minikube start --driver=docker```
- re-ran kubectl apply
- minikube service command gave URL http://192.168.49.2:30000 which brought up uptime-kuma in the browser
### Adding Volumes for Persistent Storage
```YAML
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: uptime-kuma
  labels:
    app: uptime-kuma
spec:
  replicas: 1
  selector:
    matchLabels:
      app: uptime-kuma
  template:
    metadata:
      labels:
        app: uptime-kuma
    spec:
      containers:
      - name: uptime-kuma
        image: louislam/uptime-kuma
        ports:
        - containerPort: 3001
        volumeMounts: # Volume must be created along with volumeMount (see next below)
        - name: uptime-kuma-data
          mountPath: /app/data # Path within the container, like the right side of a docker bind mount -- /tmp/data:/app/data
      volumes: # Defines a volume that uses an existing PVC (defined below)
      - name: uptime-kuma-data
        persistentVolumeClaim:
          claimName: uptime-kuma-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: uptime-kuma-service
spec:
  selector:
    app: uptime-kuma
  type: LoadBalancer  
  ports:
    - protocol: TCP
      port: 3001
      targetPort: 3001
      nodePort: 30000
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: uptime-kuma-pv
spec:
  accessModes:
    - ReadWriteOnce
  capacity:
    storage: 1Gi
  storageClassName: standard
  hostPath:
    path: /tmp/uptime/data # Path on the host
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: uptime-kuma-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  storageClassName: standard
  volumeName: uptime-kuma-pv
```

- Was confused on the need for ```storageClassName``` -- tried to comment it out and redeploy:
	- Pod was stuck in pending status
	- ```kubectl describe pod``` showed:Events:
```
  Type     Reason            Age    From               Message
  ----     ------            ----   ----               -------
  Warning  FailedScheduling  4m10s  default-scheduler  0/1 nodes are available: persistentvolumeclaim "uptime-kuma-pvc" not found. preemption: 0/1 nodes are available: 1 Preemption is not helpful for scheduling.
  Warning  FailedScheduling  4m9s   default-scheduler  0/1 nodes are available: pod has unbound immediate PersistentVolumeClaims. preemption: 0/1 nodes are available: 1 Preemption is not helpful for scheduling.
```
- ```kubectl describe pvc uptime-kuma-pvc``` showed:
```
Events:
  Type     Reason          Age                  From                         Message
  ----     ------          ----                 ----                         -------
  Warning  VolumeMismatch  7s (x26 over 6m20s)  persistentvolume-controller  Cannot bind to requested volume "uptime-kuma-pv": storageClassName does not match
```
-```kubectl describe pv uptime-kuma-pv``` shows that uptime-kuma-pv has no storage class assigned:
```
Name:            uptime-kuma-pv
Labels:          <none>
Annotations:     <none>
Finalizers:      [kubernetes.io/pv-protection]
StorageClass:    
Status:          Available
```
- The docs state the following: *You can create a PersistentVolumeClaim without specifying a `storageClassName` for the new PVC, and you can do so even when no default StorageClass exists in your cluster. In this case, the new PVC creates as you defined it, and the `storageClassName` of that PVC remains unset until a default becomes available.*
- Checking what my default storage class is with ```kubectl get storageclass``` (I didn't set this... must be a minikube default)
```
NAME                 PROVISIONER                RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
standard (default)   k8s.io/minikube-hostpath   Delete          Immediate           false                  2d
```
- **It appears that a PVC will be set to the default storage class if one is not specified, but the same is not true for a PV -- the docs also state:** *A PV with no storageClassName has no class and can only be bound to PVCs that request no particular class.*
- After understanding this I ran ```kubectl apply -f uptime-kuma-deployment.yml``` with the above configuration.
- Ran ```minikube service uptime-kuma-service``` to pull it up in the browser
- Created an uptime-kuma user and created a monitor for google.com
- Ran ```kubectl delete deployment uptime-kuma``` to delete all pods, then re-ran ```kubectl apply -f uptime-kuma-deployment```, then opened it in the browser again with ```minikube service uptime-kuma-service``` 
- My user and monitor still exist after recreating the deployment, so data was successfully persisted with a PV and PVC!  🎉🎉🎉
### Using an SMB share as persistent storage
- Config files are in the folder ```remote-storage-smb-deployment```
- Created a share on TrueNAS called 'kubernetes-testing'
- Installed the SMB CSI driver -- https://github.com/kubernetes-csi/csi-driver-smb/tree/master
- Generate a secret to store share credentials with ```kubectl create secret generic smbcreds --from-literal username=USERNAME --from-literal password="PASSWORD"```
	- note: the *generic* flag is used to denote that the secret being created is an 'Opaque' secret (arbitrary, user-defined data). There are other secret types for various specific use cases such as docker registry credentials or tls certificates. https://kubernetes.io/docs/concepts/configuration/secret/
- Used this video from the Youtube channel Jim's garage for reference with the PV and PVC configs https://www.youtube.com/watch?v=3S5oeB2qhyg&t=318s
- Got a permission denied error on the pod, appears to be when mounting smb share in fstab
- Ran the following to verify contents of the secret: ```kubectl get secret smbcreds  -o jsonpath='{.data}'``` which gave the following output: ```{"password":"MXFhQFdTM2Vk","username":"a3VzZXI="}```
- These values are base64 encoded, so I ran the password through an online decoder and found that it appears to have been cut off after 9 characters. 
- Found out the reason that it was cut off was that when using double quotes the shell will interpret special characters. Since the password was a waterfall of 1qa@WS3ed$RF5tg --> the shell interpreted everything after the dollar sign as a variable (which was obviously empty)
- After fixing this, everything appears to deploy without errors, however connecting with minikube service gives 'ERR_CONNECTION_REFUSED'
- Checking pod logs -- it appears the app can't connect to SQLLite because the database is locked
- Switched to an nginx image rather than uptme-kuma -- you wouldn't want a SQLLite DB on an SMB share anyway. 
- Gave a mountpath of /usr/share/nginx/html
- I thought if nothing existed for this volume it would create a default splashpage -- it didn't, and I got 403 forbidden
- Added a hello-world HTML file to the share and then ran ```kubectl rollout restart deployment nginx``` to restart the pod(s)
- It worked! :tada:
- Scaled pods up from 1 to 2 and re-ran kubectl apply -- still works! :tada: