# [KongHQ](https://konghq.com)

Read about [Charts](https://github.com/Kong/charts?tab=readme-ov-file#charts)

**TLDR;**

For an on‑prem Kubernetes cluster using only free components, use Kong Gateway OSS with Kong Ingress Controller, not Konnect or enterprise features.

Use this for a simple, free setup (recommended):

- kong/ingress – installs Kong Ingress Controller (KIC) + Kong Gateway (DB‑less, OSS by default). [Install KIC]. The default values install KIC in Gateway Discovery mode with a DB‑less Kong Gateway, which is the recommended topology. [Install KIC]

Alternative if you want to manage Gateway yourself (no KIC):

- kong/kong – installs Kong Gateway only (can be OSS) and you configure it via Admin API, decK, Terraform, etc. [Kong on‑prem Helm]

Do not use for your case (based on your requirements):

- kong/gateway-operator and kong/kong-operator – Kong Operator is focused on Konnect and CRD‑driven deployments, and the Konnect‑focused install is explicitly marked as incompatible with on‑prem in that guide. [Operator install]
Operator is useful if you want declarative CRDs to provision CP/DPs, but it adds complexity and isn’t required for a basic free, on‑prem setup.

So for a free, Kubernetes‑native, on‑prem deployment:

- Start with kong/ingress (KIC + OSS Gateway).
- Use kong/kong only if you prefer to manage Kong Gateway directly rather than through Kubernetes CRDs.

**Long:**

- **Data Plane**: The Kong Gateway data plane is responsible for processing incoming traffic. It receives the routing configuration from the control plane using the clustering endpoint. This is built over Nginx as the base
  - **Gateway**: This is proxy server which is node of Data plane
    - **Admin API**: Manage on-prem Kong Gateway entities via an API, url shows avaiable config syntaxes
- **Control Plane**: The control plane contains all Kong Gateway configurations. The configuration is stored in a PostgreSQL database.
  - The control plane is where operators access kong for pushing configs and fetching logs
  - Operating modes:
    - **DB Less Mode**: In this mode the entire configuration is managed in-memory loaded from a configuration file.
      - Some of the pulugins like rate limiting will not work fully in a DB less model.
    - **DB Mode (Traditional)**: persists all the configurations in a DB with a couple of choices(Cassandra / PostgreSQL). It's also important to note that kong stores all the configuration in memory for better performance. DB is reached mostly to refresh config on change. Cassandra has its own advantage of horizontal scaling to match the horizontal scalability of Kong. The fully functional mode for Kong Gateway is traditional mode (also known as DB mode). in traditional mode, you cannot push configs using Kubernetes CRDs. CRDs are used with Kong Ingress Controller or Kong Gateway Operator in Kubernetes environments, not with traditional mode.
- **Ingress Controller**: Configure Kong Gateway using Kubernetes-native resources such as httproute and ingress.

**CRDs (Custom Resource Definitions)** are used to provide a Kubernetes-native way to configure Kong Gateway. Both the Kong Ingress Controller (KIC) and the Kong Gateway Operator use CRDs, but in different ways:

- Kong Ingress Controller: KIC uses CRDs to allow you to manage Kong Gateway configuration using Kubernetes resources like KongPlugin, KongConsumer, etc. This is the standard method for configuring Kong Gateway in Kubernetes with KIC Custom Resources.
- Kong Gateway Operator: The Operator is also Kubernetes-native and is driven entirely by CRDs. It allows you to deploy and configure Kong’s products in a fully declarative way using Kubernetes resources About Kong Gateway Operator.

**NOTE** `kong/kong-operator` is new name for `kong/gateway-operator`. Github page always redirects to kong-operator for both

## Install [Kubernetes Gateway API](https://github.com/kubernetes-sigs/gateway-api) CRDs

```bash
helm repo add kong https://charts.konghq.com

# Install the Gateway API CRDs
# This command will install all resources that have graduated to GA or beta, including GatewayClass, Gateway, HTTPRoute, and ReferenceGrant.
kubectl apply --server-side -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.4.0/standard-install.yaml

# Or, if you want to use experimental resources and fields such as TCPRoutes and UDPRoutes, please run this command.
kubectl apply --server-side -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.4.0/experimental-install.yaml
```

## [Install Kong Ingress Controller (KIC)](./docs/ingress.md)

This is the primary and recommended Helm chart for deploying Kong Gateway as an Ingress Controller within a Kubernetes cluster. It includes the Kong Ingress Controller, which acts as the control plane, translating Kubernetes Ingress and other custom resources (like KongPlugin, KongConsumer, etc.) into a configuration that the Kong Gateway (data plane) can understand and apply. The kong/ingress chart essentially sets up Kong to manage inbound traffic for your Kubernetes services using Kubernetes-native APIs.

In essence: provides the complete solution for running Kong as an Ingress Controller in Kubernetes, including both the control plane (Ingress Controller) and the data plane (Kong Gateway).

**Therefore**, when deploying Kong to manage traffic in a Kubernetes environment, the kong/ingress Helm chart is the appropriate choice as it automates the integration with Kubernetes Ingress and other resources.

- The Kong Ingress Controller [configures](https://developer.konghq.com/kubernetes-ingress-controller/architecture/#architecture) Kong Gateway using Ingress or Gateway API resources created inside a Kubernetes cluster.
- KIC does not handle traffic directly; it only updates Kong Gateway’s configuration in response to changes in the Kubernetes cluster.
- CRDs are used with Kong Ingress Controller or Kong Gateway Operator in Kubernetes environments, not with traditional mode.

## [Deploy Kong Gateway on Kubernetes](./docs/gw.md)

This chart is a dependency of the kong/ingress chart. It specifically deploys the Kong Gateway itself, which is the data plane responsible for proxying and managing API traffic. While you could theoretically deploy just the kong/kong chart, it would require manual configuration of Kong Gateway, as it wouldn't be integrated with the Kubernetes Ingress mechanism without the Kong Ingress Controller.

In essence: provides only the Kong Gateway data plane, and is leveraged by the kong/ingress chart to deploy the core proxy functionality.

Kong Gateway is the API gateway that proxies and manages traffic.

## REFERENCE

- Kong Gateway
  - [All Gateway docs](https://developer.konghq.com/index/gateway/)
  - [All KIC docs](https://developer.konghq.com/index/kubernetes-ingress-controller/)
- [Kong Ingress Controller](https://developer.konghq.com/kubernetes-ingress-controller/)
- [Plugins](https://developer.konghq.com/plugins/)
  - [Kong Gateway entities: Plugins](https://developer.konghq.com/gateway/entities/plugin/)
  - [Kong Gateway Plugins Compatibility](https://developer.konghq.com/plugins/compatibility/#site-base-gateway-plugins-compatibility)
- [Tools](https://developer.konghq.com/tools/)
  - [decK CLI](https://developer.konghq.com/deck/): Manage Kong Gateway configuration through declarative state files
  - [terraform](https://developer.konghq.com/terraform/)
- [Kong Manager configuration](https://developer.konghq.com/gateway/kong-manager/configuration/#enable-kong-manager/)
- [Upgrade](https://developer.konghq.com/kubernetes-ingress-controller/faq/upgrading-gateway/)
