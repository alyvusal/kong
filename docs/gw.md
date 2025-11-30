# Kong Gateway

## [Helm](https://developer.konghq.com/gateway/install/kubernetes/on-prem/)

Platform: on-prem

```bash
kubectl create namespace kong

# create database to streo configs (no need for DB-less mode)
helm upgrade -i kong-cp-postgresql bitnami/postgresql \
  -n kong \
  --version 16.7.21 \
  -f k8s/helm/bitnami-postgresql.yaml

# use for enterprise edition
kubectl -n kong create secret generic kong-enterprise-license --from-file=license=license.json

# Kong Gateway uses mTLS to secure the control plane/data plane communication when running in hybrid mode.
openssl req -new -x509 -nodes -newkey ec:<(openssl ecparam -name secp384r1) \
  -keyout /tmp/tls.key -out /tmp/tls.crt -days 1095 -subj "/CN=kong_clustering"

# Create a Kubernetes secret containing the certificate.
kubectl -n kong create secret tls kong-cluster-cert --cert=/tmp/tls.crt --key=/tmp/tls.key

# Create a Control Plane
# The control plane contains all Kong Gateway configurations. The configuration is stored in a PostgreSQL database.
helm upgrade -i kong-cp kong/kong \
  -n kong \
  --version 2.52.0 \
  -f k8s/helm/kong-cp.yaml

# Create a Data Plane
# The Kong Gateway data plane is responsible for processing incoming traffic. It receives the routing configuration from the control plane using the clustering endpoint.
helm upgrade -i kong-dp kong/kong \
  -n kong \
  --version 2.52.0 \
  -f k8s/helm/kong-dp.yaml
```

- Kong Manager is the graphical user interface (GUI) for Kong Gateway. It uses the Kong Admin API under the hood to administer and control Kong Gateway.
- Kong’s Admin API must be accessible over HTTP from your local machine to use Kong Manager

Open manager UI: [http://manager-172.18.0.2.nip.io](http://manager-172.18.0.2.nip.io)

- username: kong_admin
- password: {{ env.password }} in values file

Check doc for EKS, AKS, GKE settings. In helm chart we used only KIC method:

- [Configure the Admin API with Kong Gateway on Kubernetes](https://developer.konghq.com/gateway/install/kubernetes/on-prem/admin/)
- [Enable Kong Manager with Kong Gateway on Kubernetes](https://developer.konghq.com/gateway/install/kubernetes/on-prem/manager/)

### [DB-less mode](https://developer.konghq.com/gateway/db-less-mode/)

You do not need a Control Plane in DB-less mode. In DB-less mode, each Kong Gateway node operates independently, loading its configuration from a declarative YAML or JSON file directly into memory, without any central database, KIC CRDs or Control Plane. All configuration changes must be made by updating and reloading this file on each node. There is no central coordination or propagation of configuration between nodes in DB-less mode, and the Admin API is read-only for entity management in this mode DB-less mode DB-less mode limitations Deployment topologies.

So, in this mode: Ingress, KongIngress, KongPlugin etc will not be supported, otherwise KIC will be brain of kong.yml config file. But in this mode only way is update config file. After making changes, you must reload or restart Kong Gateway to apply the new configuration.

[Kong Gateway has an official declarative configuration format](https://developer.konghq.com/deck/#kong-gateway-already-has-built-in-declarative-configuration-do-i-still-need-deck). Kong Gateway can generate such a file with the `kong config db_export` command, which exports most of Kong Gateway’s database into a file.

You can use a file in this format to configure Kong Gateway when it is running in a DB-less or in-memory mode. If you’re using Kong Gateway in DB-less mode, you can’t use decK for any sync, dump, or similar operations, as they require write access to the Admin API.

- [DB-less mode with Kong Ingress Controller](https://developer.konghq.com/gateway/db-less-mode/#db-less-mode-with-kong-ingress-controller)

**DB-less mode** is recommended when:

- You want to reduce dependencies and avoid managing a database.
- Your configuration can fit entirely in memory.
- You prefer automation and CI/CD workflows, as configuration is managed declaratively in a single file (YAML/JSON) and can be version-controlled.
- You do not require plugins or features that depend on a database, and you are comfortable with the Admin API being read-only (no CRUD operations except for - configuration reloads).
- You do not need Kong Manager for entity management, as it is not fully compatible with DB-less mode.
- However, DB-less mode has limitations, such as no central coordination between nodes, limited plugin compatibility, and the need to manually update configuration on each node. It is a good fit for simpler, stateless, and highly automated environments, but not for all production scenarios DB-less mode limitations.

**Database-backed mode (Traditional or Hybrid)** is recommended when:

- You need features or plugins that require a database (e.g., rate limiting in cluster mode, OAuth2).
- You want to use Kong Manager for managing entities.
- You require dynamic configuration changes via the Admin API.
- You need central coordination and easier management of large, complex configurations.
- Kong Enterprise documentation recommends Hybrid mode (a form of database-backed deployment) for most production scenarios, as it separates the control plane and data plane, making the platform more reliable and scalable Customer onboarding.

**Summary**:

- Use DB-less mode for simple, stateless, and highly automated production environments where its limitations are acceptable.
- Use database-backed mode (preferably Hybrid) for more complex, dynamic, or feature-rich production environments.

Use Manager UI to create yaml config and copy to kong.yml

```bash
helm upgrade -i kong-dp-dbless kong/kong \
  -n kong \
  --version 2.52.0 \
  -f k8s/helm/kong-dp-dbless.yaml


# Kong Gateway uses mTLS to secure the control plane/data plane communication when running in hybrid mode.
openssl req -new -x509 -nodes -newkey ec:<(openssl ecparam -name secp384r1) \
  -keyout /tmp/tls.key -out /tmp/tls.crt -days 1095 -subj "/CN=kong_clustering"

# Create a Kubernetes secret containing the certificate.
kubectl -n kong create secret tls kong-cluster-cert --cert=/tmp/tls.crt --key=/tmp/tls.key

# Generate sample config file (generates a file named kong.yml), login to pod
kong config init /tmp/kong.yml

# Validate config
kong config parse /tmp/kong.yml

# check content and use as sample
cat /tmp/kong.yml

# Apply your declarative kong config file
kubectl create configmap kong-config --from-file=kong.yml=k8s/kong.yml --dry-run=client -o yaml | kubectl apply -n kong -f -
# kubectl apply -f k8s/kong-config-dbless.yaml

# After applying new file
kubectl rollout restart deployment kong-dp-dbless-kong
```

### DB-less mode with Kong Ingress Controller

```bash
helm upgrade -i kong kong/kong \
  -n kong --create-namespace \
  --version 2.52.0 \
  -f k8s/helm/kong-dbless-aio.yaml
```

## [Operator](https://developer.konghq.com/operator/dataplanes/get-started/hybrid/install/)

Platform: konnect (Kong’s SaaS API platform, built on top of Kong Gateway)

[kong/gateway-operator](https://docs.konghq.com/gateway-operator/latest/) is a Kubernetes Operator that can manage your Kong Ingress Controller, Kong Gateway Data Planes, or both together when running on Kubernetes.
