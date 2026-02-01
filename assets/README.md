# Install k8s Cluster Assets

Tools:

- [Talos CLI](https://formulae.brew.sh/formula/talosctl)
- [kubectl](https://formulae.brew.sh/formula/kubernetes-cli)
- [Helm](https://formulae.brew.sh/formula/helm)
- [cilium CLI](https://formulae.brew.sh/formula/cilium-cli)

## Expose example https service

Prerequisites:

1. Public domain (e.g. `amf-cluster.duckdns.org`) with a DNS provider (e.g. duckdns.org) pointing to the router public IP address
1. NAT rule in the firewall/router forwarding the ports 80 and 443 to the MetalLB IP address (IP address defined in the `01-ip-address-pool.yaml` file)

```bash
# Set right context
alias k="kubectl"
k config use-context <kube-context>

# Install MetalLB (v0.15.3 at December 2025)
k apply -f https://raw.githubusercontent.com/metallb/metallb/v0.15.3/config/manifests/metallb-native.yaml

# Create MetalLB IP addresses pool and layer 2 advertisement
k apply -f 01-ip-address-pool.yaml

# Check MetalLB
k apply -f 02-lb-checker.yaml
k get services -n lb-checker -o wide # Check that the EXTERNAL-IP is available
curl -X GET http://<EXTERNAL-IP>:8888/
k delete -f 02-lb-checker.yaml

# Deploy and check an example application (NOTE: you should see the output: "App is running!")
k apply -f 03-example-application.yaml
k -n example-app-namespace run curl --image=curlimages/curl --rm -it --restart=Never -- curl http://example-app-service:80/

# Install Gateway API CRDs
k apply -f https://raw.githubusercontent.com/kubernetes-sigs/gateway-api/v1.2.0/config/crd/standard/gateway.networking.k8s.io_gatewayclasses.yaml && \
k apply -f https://raw.githubusercontent.com/kubernetes-sigs/gateway-api/v1.2.0/config/crd/standard/gateway.networking.k8s.io_gateways.yaml && \
k apply -f https://raw.githubusercontent.com/kubernetes-sigs/gateway-api/v1.2.0/config/crd/standard/gateway.networking.k8s.io_httproutes.yaml && \
k apply -f https://raw.githubusercontent.com/kubernetes-sigs/gateway-api/v1.2.0/config/crd/standard/gateway.networking.k8s.io_referencegrants.yaml && \
k apply -f https://raw.githubusercontent.com/kubernetes-sigs/gateway-api/v1.2.0/config/crd/standard/gateway.networking.k8s.io_grpcroutes.yaml

# Install experimental Gateway API CRDs
k apply -f https://raw.githubusercontent.com/kubernetes-sigs/gateway-api/v1.2.0/config/crd/experimental/gateway.networking.k8s.io_tlsroutes.yaml && \
k apply -f https://raw.githubusercontent.com/kubernetes-sigs/gateway-api/v1.2.0/config/crd/experimental/gateway.networking.k8s.io_tcproutes.yaml

k get crd | grep gateway

# Gateway API CRDs installation alternatives:
# k apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.2.1/standard-install.yaml
# k apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.2.1/experimental-install.yaml

# Install cert-manager with Gateway API support
helm repo add jetstack https://charts.jetstack.io
helm repo update

k apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.13.0/cert-manager.crds.yaml

helm upgrade --install cert-manager oci://quay.io/jetstack/charts/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --set config.apiVersion="controller.config.cert-manager.io/v1alpha1" \
  --set config.kind="ControllerConfiguration" \
  --set config.enableGatewayAPI=true

k get pods -n cert-manager

# Configure ACME cluster issuer
k apply -f 04-letsencrypt-cluster-issuer.yaml

# Create Gateway Class, Gateway and HTTPRoutes
k apply -f 05-gateway-class.yaml
k apply -f 06-gateway.yaml
k apply -f 07-http-route.yaml
k apply -f 08-https-route.yaml
```

## Install observability stack

```bash
# Install Prometheus Stack Community
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm upgrade --install prometheus-stack prometheus-community/kube-prometheus-stack \
  --namespace observability \
  --create-namespace
```

## Install ArgoCD

```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update
helm upgrade --install argocd argo/argo-cd \
  --namespace argocd \
  --create-namespace \
  -f 100-argocd-values.yaml

k get pods -n argocd
k get svc -n argocd
k -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo

k port-forward -n argocd service/argocd-server 8080:80
# Access https://localhost:8080 with admin/copied-password credentials
```

## NOTES

- Check why the proxmox node is not reachable when curl the EXTERNAL-IP provided by MetalLB (load balancer)
  - Maybe due to the interface configuration in the Talos nodes?
- Once the proxmox node is reachable, replace the type of the service in the example application from ClusterIP to LoadBalancer to test the full path (ingress controller + cert-manager + example application)
