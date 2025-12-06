# Talos Linux

Talos Linux is a modern, secure, and immutable operating system designed specifically for running Kubernetes clusters. This repository provides configuration files and instructions to set up a single-node Talos cluster on Proxmox.

## Resources

- [Talos Linux Official Documentation](https://docs.siderolabs.com/talos/v1.11/platform-specific-installations/virtualized-platforms/proxmox)
- [Deploy Cilium CLI on Talos](https://cilium.io/blog/2023/09/28/cilium-cli-talos/)
- [Having fun with Cilium (BGP)](https://medium.com/@rob.de.graaf88/having-fun-with-cilium-bgp-talos-and-unifi-cloud-gateway-ultra-111ffb39757e)

## Single Node Cluster Setup Guide on Proxmox

1. Install talosctl (`brew install talosctl`)
2. Create Proxmox VM
3. Mount Talos ISO and boot from it (Talos ISO can be downloaded from the [Talos Linux Image Factory](https://factory.talos.dev/))
4. Find the IP address of the Talos node (console output)
5. Set the Talos node IP address:
   ```bash
   export TALOSCTL_NODE=<Talos_Node_IP>
   ```
6. Generate secrets for the Talos cluster:
   ```bash
   talosctl gen secrets
   ```
7. Generate Talos configuration files (IMPORTANT: make sure to customize the `talos-single-node.yaml`):
   ```bash
   talosctl gen config <cluster-name> https://$TALOSCTL_NODE:6443 \
     --with-secrets secrets.yaml \
     --config-patch @talos-single-node.yaml \
     --output output_config
   ```
8. Apply the Talos configuration to the node:
   ```bash
   talosctl apply-config --insecure --nodes $TALOSCTL_NODE --file output_config/talosconfig.yaml
   ```
9. Reboot the Talos node to apply the configuration:
   ```bash
   talosctl reboot --insecure --nodes $TALOSCTL_NODE
   ```
10. Monitor the node status until it becomes ready:
    ```bash
    talosctl --insecure --nodes $TALOSCTL_NODE get nodes
    ```
11. Once the node is ready, you can interact with the Kubernetes cluster using `kubectl`:
    ```bash
    export KUBECONFIG=$(talosctl kubeconfig --insecure --nodes $TALOSCTL_NODE)
    kubectl get nodes
    ```

## Creating Helm template for Talos Cilium Installation

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
