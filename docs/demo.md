# Demo

## DB-less mode

```bash
# Run kind lb tool, it will assign docker ip to Loadbalancer type services
cloud-provider-kind

# Fetch the LoadBalancer address for the kong-dp service and store it in the PROXY_IP environment variable:
PROXY_IP=$(kubectl get service -n kong kong-dp-kong-proxy -o jsonpath='{range .status.loadBalancer.ingress[0]}{@.ip}{@.hostname}{end}')

# Test by path
curl 172.18.0.2/httpbin/ip

# Test by hostname
curl -XPUT hello-172.18.0.2.nip.io/put
```

## Mock Route

```bash
# Run kind lb tool, it will assign docker ip to Loadbalancer type services
cloud-provider-kind

# or forward kong proxy service
kubectl port-forward svc/kong-dp-kong-proxy 9080:80

# Fetch the LoadBalancer address for the kong-dp service and store it in the PROXY_IP environment variable:
PROXY_IP=$(kubectl get service -n kong kong-dp-kong-proxy -o jsonpath='{range .status.loadBalancer.ingress[0]}{@.ip}{@.hostname}{end}')

# Make an HTTP request to your $PROXY_IP. This will return a HTTP 404 served by Kong Gateway:
curl $PROXY_IP/mock/anything  # or curl -i 127.0.0.1:9080/mock/anything

# Create a mock Service and Route:
kubectl port-forward -n kong service/kong-cp-kong-admin 8001
curl localhost:8001/services -d name=mock -d url="https://httpbin.konghq.com"
curl localhost:8001/services/mock/routes -d "paths=/mock"
# or
curl admin-172.18.0.2.nip.io/services -d name=mock -d url="https://httpbin.konghq.com"
curl admin-172.18.0.2.nip.io/services/mock/routes -d "paths=/mock"

# Make an HTTP request to your $PROXY_IP again. This time Kong Gateway will route the request to httpbin:
curl $PROXY_IP/mock/anything  # or curl -i 127.0.0.1:9080/mock/anything
```

## [Services and Routes](https://developer.konghq.com/kubernetes-ingress-controller/get-started/services-and-routes/)

```bash
# Fetch the LoadBalancer address for the kong-dp service and store it in the PROXY_IP environment variable:
export PROXY_IP=$(kubectl get svc -n kong kong-ingress-gateway-proxy -o jsonpath='{range .status.loadBalancer.ingress[0]}{@.ip}{@.hostname}{end}')

# install app
kubectl apply -f examples/echo-service/echo-service.yaml # kubectl apply -f https://developer.konghq.com/manifests/kic/echo-service.yaml

# Expose over httproute
kubectl apply -f examples/echo-service/ingress-enable-gateway-api.yaml  # Enable Gateway API (optional), needed for HTTPRoute, GatewayClass and Gateway
kubectl apply -f examples/echo-service/HTTPRoute.yaml  # kubectl apply -f  kubectl apply -f https://developer.konghq.com/manifests/kic/echo-service.yaml
# or
kubectl apply -f examples/echo-service/Ingress.yaml

curl "$PROXY_IP/echo"
```

## [Rate Limiting](https://developer.konghq.com/kubernetes-ingress-controller/get-started/rate-limiting/)

[Rate limiting plugin](https://developer.konghq.com/plugins/rate-limiting/) is used to control the rate of requests sent to an upstream service. It can be used to prevent DoS attacks, limit web scraping, and other forms of overuse. Without rate limiting, clients have unlimited access to your upstream services, which may negatively impact availability. ([The Rate Limiting Advanced plugin](https://developer.konghq.com/plugins/rate-limiting-advanced/) is also available. The advanced version provides additional features such as support for the sliding window algorithm and advanced Redis support for greater performance.)

```bash
# applied to a specific service or route. Set config.minute to the number of requests allowed per minute.
kubectl apply -f examples/echo-service/KongPlugin-RateLimit.yaml

# apply the KongPlugin resource by annotating the httproute or ingress resource:
kubectl annotate httproute echo konghq.com/plugins=rate-limit-5-min  # kubectl annotate ingress echo konghq.com/plugins=rate-limit-5-min

for _ in {1..6}; do
  curl -i $PROXY_IP/echo -H "apikey:example-key"; echo
done
```

## [Proxy Caching](https://developer.konghq.com/kubernetes-ingress-controller/get-started/proxy-caching/)

The proxy-cache plugin returns a X-Cache-Status header that can contain the following cache results:

Kong Gateway identifies the status of a request’s proxy cache behavior via the X-Cache-Status header. There are several possible values for this header:

- `Miss`: The request could be satisfied in cache, but an entry for the resource was not found in cache, and the request was proxied upstream.
- `Hit`: The request was satisfied and served from cache.
- `Refresh`: The resource was found in cache, but couldn’t satisfy the request, due to Cache-Control behaviors or from reaching its hardcoded config.cache_ttl threshold.
- `Bypass`: The request couldn’t be satisfied from cache based on plugin configuration.

```bash
# global plugin that applies to all services.
kubectl apply -f examples/echo-service/KongClusterPlugin-Caching.yaml


for _ in {1..6}; do
  curl -sv $PROXY_IP/echo -H "apikey:example-key" 2>&1 | grep -E "(Status|< HTTP)"; echo
done
```

The first request results in X-Cache-Status: Miss. This means that the request is sent to the upstream service. The next four responses return X-Cache-Status: Hit which indicates that the request was served from a cache. If you receive a HTTP 429 from the first request, wait 60 seconds for the rate limit timer to reset.

## [Key Authentication](https://developer.konghq.com/kubernetes-ingress-controller/get-started/key-authentication/)

Key authentication in Kong Gateway works by using the Consumer entity. Keys are assigned to Consumers, and client applications present the key within the requests they make.

```bash
# Create a KongPlugin resource containing an authentication plugin configuration and annotate your Kubernetes service with the plugin name
kubectl apply -f examples/echo-service/KongPlugin-key-auth.yaml

# kubectl annotate service YOUR_SERVICE konghq.com/plugins=key-auth
kubectl annotate service echo konghq.com/plugins=rate-limit-5-min,key-auth --overwrite

# This request returns a 401 error with the message Unauthorized.
curl -i $PROXY_IP/echo

# Set up Consumers and keys
# Create a new Secret labeled to use key-auth credential type:
kubectl apply -f examples/echo-service/secret-key-auth.yaml
# Create a new Consumer and attach the credential
kubectl apply -f examples/echo-service/Consumer-key-auth.yaml

curl "$PROXY_IP/echo" -H "apikey:hello_world"

for _ in {1..6}; do
  curl -i $PROXY_IP/echo -H "apikey:hello_world"; echo
done
```
