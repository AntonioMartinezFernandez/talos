# Talos Linux

Talos Linux is a modern, secure, and immutable operating system designed specifically for running Kubernetes clusters. This repository provides configuration files and instructions to set up a single-node Talos cluster on Proxmox.

Tools:

- [Talos CLI](https://formulae.brew.sh/formula/talosctl)
- [kubectl](https://formulae.brew.sh/formula/kubernetes-cli)
- [Helm](https://formulae.brew.sh/formula/helm)
- [cilium CLI](https://formulae.brew.sh/formula/cilium-cli)

## Resources

- [Install k8s Cluster Assets](./assets/README.md)

- [Talos Linux Official Documentation](https://docs.siderolabs.com/talos/v1.11/platform-specific-installations/virtualized-platforms/proxmox)
- [Deploy Cilium CLI on Talos](https://cilium.io/blog/2023/09/28/cilium-cli-talos/)
- [Having fun with Cilium (BGP)](https://medium.com/@rob.de.graaf88/having-fun-with-cilium-bgp-talos-and-unifi-cloud-gateway-ultra-111ffb39757e)

### Creating Helm template for Talos Cilium Installation

With this command you can create a Helm template for installing Cilium on Talos, which you can then reference in the Talos configuration file (`talos-cluster-config.yaml`):

```bash
helm repo add cilium https://helm.cilium.io/
helm repo update

helm template \
  cilium \
  cilium/cilium \
  --version 1.18.6 \
  --namespace kube-system \
  --set ipam.mode=kubernetes \
  --set kubeProxyReplacement=true \
  --set securityContext.capabilities.ciliumAgent="{CHOWN,KILL,NET_ADMIN,NET_RAW,IPC_LOCK,SYS_ADMIN,SYS_RESOURCE,DAC_OVERRIDE,FOWNER,SETGID,SETUID}" \
  --set securityContext.capabilities.cleanCiliumState="{NET_ADMIN,SYS_ADMIN,SYS_RESOURCE}" \
  --set cgroup.autoMount.enabled=false \
  --set cgroup.hostRoot=/sys/fs/cgroup \
  --set k8sServiceHost=localhost \
  --set kubeProxyReplacement=true \
  --set gatewayAPI.enabled=true \
  --set k8sServicePort=7445 > cilium-gateway-api-helm-template.yaml
```

## Control Plane Setup Guide (using VM on Proxmox)

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
7. Generate Talos configuration files (IMPORTANT: make sure to customize the `talos-cluster-config.yaml` before running this command):
   ```bash
   talosctl gen config talos-proxmox-cluster https://$MASTER_NODE_IP:6443 \
     --with-secrets secrets.yaml \
     --config-patch @talos-cluster-config.yaml \
     --output output_config
   ```
8. In the `output_config/controlplane.yaml` file, update the fields:
   ```
   machine.network.hostname: <desired-hostname-for-controlplane> - e.g.: talos-proxmox (or define it on v1alpha1/HostnameConfig CR at the bottom)
   ```
9. Apply the Talos configuration to the control plane (the only node in this case):
   ```bash
   talosctl apply-config --insecure --nodes $DHCP_MASTER_IP --file output_config/controlplane.yaml
   ```
10. Wait until the node reboots and gets the desired IP address

## Worker Node Setup Guide (using Raspberry Pi)

1. Update the Raspberry Pi EPROM to the latest version using the [Raspberry Pi Imager](https://www.raspberrypi.com/software/) and selecting Misc utility images under the Operating System tab. Power the Raspberry Pi on with the SD card recently flashed, and wait at least 10 seconds. If successful, the green LED light will blink rapidly (forever), otherwise an error pattern will be displayed. If an HDMI display is attached to the port closest to the power/USB-C port, the screen will display green for success or red if a failure occurs.
1. Download the Talos image for Raspberry Pi (check last version at [Talos Linux Image Factory](https://factory.talos.dev/))
   ```bash
   curl -LO https://factory.talos.dev/image/ee21ef4a5ef808a9b7484cc0dda0f25075021691c8c09a276591eedb638ea1f9/v1.11.5/metal-arm64.raw.xz
   xz -d metal-arm64.raw.xz
   ```
1. Flash the Talos image to the SD card
   ```bash
   sudo diskutil list # Identify your SD card (e.g., /dev/disk4)
   sudo diskutil eraseDisk FAT32 raspi MBRFormat /dev/<diskN> # Replace <diskN> with your SD card identifier
   sudo diskutil unmountDisk /dev/<diskN> # Replace <diskN> with your SD card identifier
   sudo dd if=metal-arm64.raw of=/dev/<diskN> conv=fsync bs=4M # Replace <diskN> with your SD card identifier
   ```
1. Insert the SD card into the Raspberry Pi and power it on
1. Find the IP address of the Talos node (console output or check your router's DHCP leases)
1. Get the disk identifier for the Raspbberry Pi (usually `/dev/mmcblk0`) and the network interface name (usually `end0`):
   ```bash
   talosctl apply-config --insecure --mode=interactive --nodes <raspberry-pi-ip>
   ```
1. In the `output_config/worker.yaml` file, update the fields:

```
machine.network.hostname: <desired-hostname-for-raspberry-pi> - e.g.: talos-raspi-1 (or define it on v1alpha1/HostnameConfig CR at the bottom)
machine.network.interfaces[0].interface: end0 # Ensure this matches the interface name found earlier
machine.network.interfaces[0].addresses: <desired-ip-address-for-raspberry-pi>/24 - e.g.: 192.168.1.191/24
machine.install.disk: /dev/mmcblk0 # Ensure this matches the disk identifier found earlier
```

1. Bootstrap the node to join the cluster:
   ```bash
   talosctl apply-config --insecure --nodes <raspberry-pi-ip> --file output_config/worker.yaml
   ```

## Util commands for Talos Cluster Management

Commands to bootstrap and manage the Talos cluster:

```bash
export TALOSCONFIG=output_config/talosconfig # Configure talosctl to use the generated talosconfig
cp output_config/talosconfig ~/.talos/config # Optional: copy talosconfig to default location

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

# Example:
talosctl --context talos-proxmox-cluster apply-config -f output_config/controlplane.yaml -n 192.168.1.190
```

For testing purposes, you can run an nginx pod:

```bash
kubectl run nginx --image=nginx --port=80
kubectl port-forward pod/nginx 8080:80

curl http://localhost:8080

kubectl delete pod/nginx
```

## Update Talos cluster

```bash
# Upgrade Talos version
talosctl --context talos-proxmox-cluster upgrade -n <CONTROL_PLANE_IP_ADDRESS>
talosctl --context talos-proxmox-cluster upgrade -n <NODE_IP_ADDRESS>

# Upgrade k8s version
talosctl --context talos-proxmox-cluster upgrade-k8s -n <CONTROL_PLANE_IP_ADDRESS>
```
