# Talos Linux

Talos Linux is a modern, secure, and immutable operating system designed specifically for running Kubernetes clusters. This repository provides configuration files and instructions to set up a single-node Talos cluster on Proxmox.

## Resources

- [Talos Linux Official Documentation](https://docs.siderolabs.com/talos/v1.11/platform-specific-installations/virtualized-platforms/proxmox)
- [Deploy Cilium CLI on Talos](https://cilium.io/blog/2023/09/28/cilium-cli-talos/)
- [Having fun with Cilium (BGP)](https://medium.com/@rob.de.graaf88/having-fun-with-cilium-bgp-talos-and-unifi-cloud-gateway-ultra-111ffb39757e)

## Single Node Cluster Setup Guide on Proxmox

1. Install talosctl (`brew install talosctl`)
2. Create Proxmox VM with the following specifications:
   - OS Type: Linux
   - Disk Size: 20 GB
   - CPU: 4 cores
   - RAM: 4096 MB
   - Network: Bridged mode (ensure it can get an IP via DHCP) (\*) You can also set a passthrough NIC if needed
3. Mount Talos ISO and boot from it (Talos ISO can be downloaded from the [Talos Linux Image Factory](https://factory.talos.dev/))
4. Find the IP address of the Talos node (console output)
5. Set the Talos node IP address:
   ```bash
   export DHCP_MASTER_IP=<ip-address-from-console>
   export MASTER_NODE_IP=<desired-talos-node-ip>
   ```
6. Generate secrets for the Talos cluster:
   ```bash
   talosctl gen secrets
   ```
7. Generate Talos configuration files (IMPORTANT: make sure to customize the `talos-single-node-cluster.yaml` before running this command):
   ```bash
   talosctl gen config talos-proxmox-cluster https://$MASTER_NODE_IP:6443 \
     --with-secrets secrets.yaml \
     --config-patch @talos-single-node-cluster.yaml \
     --output output_config
   ```
8. Apply the Talos configuration to the control plane (the only node in this case):
   ```bash
   talosctl apply-config --insecure --nodes $DHCP_MASTER_IP --file output_config/controlplane.yaml
   ```
9. Wait until the node reboots and gets the desired IP address

## Util commands for Talos Cluster Management

Commands to bootstrap and manage the Talos cluster:

```bash
export TALOSCONFIG=output_config/talosconfig

talosctl config contexts # List contexts
talosctl config endpoint $MASTER_NODE_IP # Define the endpoint
talosctl bootstrap -n $MASTER_NODE_IP # Bootstrap the cluster
talosctl kubeconfig -n $MASTER_NODE_IP # Get kubeconfig and put it in ~/.kube/config
talosctl dashboard -n $MASTER_NODE_IP # Access Talos dashboard

talosctl -n <node-ip> reset # (*) Remove a node from the cluster (if needed)
```

To update the configuration of the node (not use `--insecure`):

```bash
talosctl apply-config -f output_config/controlplane.yaml -n $MASTER_NODE_IP
```

For testing purposes, you can run an nginx pod:

```bash
kubectl run nginx --image=nginx --port=80
kubectl port-forward pod/nginx 8080:80

curl http://localhost:8080

kubectl delete pod/nginx
```

## Creating Helm template for Talos Cilium Installation

With this command you can create a Helm template for installing Cilium on Talos, which you can then reference in the Talos configuration file (`talos-single-node-cluster.yaml`):

```bash
helm repo add cilium https://helm.cilium.io/
helm repo update

helm template \
  cilium \
  cilium/cilium \
  --version 1.18.0 \
  --namespace kube-system \
  --set ipam.mode=kubernetes \
  --set kubeProxyReplacement=true \
  --set securityContext.capabilities.ciliumAgent="{CHOWN,KILL,NET_ADMIN,NET_RAW,IPC_LOCK,SYS_ADMIN,SYS_RESOURCE,DAC_OVERRIDE,FOWNER,SETGID,SETUID}" \
  --set securityContext.capabilities.cleanCiliumState="{NET_ADMIN,SYS_ADMIN,SYS_RESOURCE}" \
  --set cgroup.autoMount.enabled=false \
  --set cgroup.hostRoot=/sys/fs/cgroup \
  --set k8sServiceHost=localhost \
  --set k8sServicePort=7445 > cilium-helm-template.yaml
```
