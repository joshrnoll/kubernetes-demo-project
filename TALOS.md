## Creating a Talos Cluster
https://www.talos.dev/v1.9/introduction/getting-started/
- Needed to install brew on my machine
- Installed talosctl:
```brew install siderolabs/tap/talosctl```
- Downloaded the ISO and booted two virtual machines from it in Proxmox
- Got the IP address of one of the nodes -- ex. 192.168.0.100
- Generated talos config using talosctl:
```talosctl gen config talos-demo https://192.168.0.100:6443```
- This produced three files in my home directory:
```
/home/josh/talosconfig
/home/josh/controlplane.yaml
/home/josh/worker.yaml
```
- Applied control plane config to the first node:
```
talosctl apply-config --insecure --nodes 192.168.0.100 --file /home/josh/controlplane.yaml
```
- Bootstrapped the cluster with this command:
```
talosctl bootstrap --nodes 192.168.0.100 --endpoints 192.168.0.100 --talosconfig=./talosconfig
```
- Once the cluster was bootstrapped, I downloaded the kubeconfig with this command:
```
talosctl kubeconfig --nodes 192.168.0.100 --endpoints 192.168.0.100 --talosconfig=./talosconfig
```
- added the second VM as a worker node to the cluster with the following command:
```
talosctl apply-config --insecure --nodes 192.168.0.101 --file /home/josh/worker.yaml
```
Finally, ```kubectl get nodes``` shows:
```
NAME            STATUS   ROLES           AGE   VERSION
talos-2sj-juj   Ready    control-plane   23h   v1.32.0
talos-jd1-prm   Ready    <none>          23h   v1.32.0
```
