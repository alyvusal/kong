# Kong  Ingress Controller (KIC)

Kong Ingress Controller's primary purpose is to satisfy Ingress resources created in k8s. It uses CRDs for more fine grained control over routing and for Kong specific configuration.

## [Helm](https://developer.konghq.com/kubernetes-ingress-controller/install/)

Platform: on-prem

```bash
helm upgrade -i kong-ingress kong/ingress \
  -n kong --create-namespace \
  --version 0.21.0 \
  -f k8s/helm/ingress.yaml
```

## Operator

Platform: on-prem

```bash
# CRDs
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.3.0/standard-install.yaml

helm upgrade -i kgo kong/gateway-operator \
  -n kong --create-namespace \
  --version 0.6.1 \
  -f k8s/helm/gateway-operator.yaml

kubectl apply -f k8s/kgo-config.yaml

kubectl get -n kong gateway kong -o wide

# If the Gateway has Programmed condition set to True, you can visit Konnect and see your configuration being synced by Kong Ingress Controller.
kubectl get -n kong gateway kong -o=jsonpath='{.status.conditions[?(@.type=="Programmed")]}' | jq
```
