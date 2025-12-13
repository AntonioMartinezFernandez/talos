# Install Talos Cluster Assets

Prerequisites:

1. Public domain (e.g. `amf-cluster.duckdns.org`) with a DNS provider (e.g. duckdns.org) pointing to the router public IP address
1. NAT rule in the firewall/router forwarding the ports 80 and 443 to the MetalLB IP address (IP address defined in the `01-ip-addresses.yaml` file)

```bash
# Install Prometheus Stack Community
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install prometheus-stack prometheus-community/kube-prometheus-stack --namespace monitoring --create-namespace

# Install MetalLB (v0.15.3 at December 2025)
k apply -f https://raw.githubusercontent.com/metallb/metallb/v0.15.3/config/manifests/metallb-native.yaml

# Create MetalLB IP addresses pool and layer 2 advertisement
k apply -f 01-ip-address-pool.yaml

# Check MetalLB
k apply -f 02-lb-checker.yaml
k get services -n lb-checker -o wide # Check that the EXTERNAL-IP is available
curl -X GET http://<EXTERNAL-IP>:8888/
k delete -f 02-lb-checker.yaml

# Install the Ingress Controller (v1.14.1 at December 2025) - Check latest nginx ingress controller version for cloud (not for baremetal)
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.14.1/deploy/static/provider/cloud/deploy.yaml

# Check the ingress-nginx-controller have an external IP address (EXTERNAL-IP)
k -n ingress-nginx get svc ingress-nginx-controller

# Install cert-manager
helm repo add jetstack https://charts.jetstack.io
helm repo update
helm install \
 cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --set crds.enabled=true

# Check cert-manager pods
kubectl -n cert-manager get pods

# Create a ClusterIssuer for cert-manager
k apply -f 03-letsencrypt-cluster-issuer.yaml

# Deploy an example application
k apply -f 04-example-application.yaml
```

## NOTES

- Check why the proxmox node is not reachable when curl the EXTERNAL-IP provided by MetalLB (load balancer)
  - Maybe due to the interface configuration in the Talos nodes?
- Once the proxmox node is reachable, replace the type of the service in the example application from ClusterIP to LoadBalancer to test the full path (ingress controller + cert-manager + example application)
