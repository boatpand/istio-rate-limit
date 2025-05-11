## Install Istio

```sh
kustomize build  istio --enable-helm | kubectl apply -f -
```

## Local rate limit

limit at envoy proxy level (per pod)

```sh
kubectl label ns local-rate-limit istio-injection=enabled
```

```sh
kubeclt apply -f local-rate-limit/container-registry-secrets.yaml
```

```sh
kubeclt apply -f local-rate-limit/nginx.yaml
```

```sh
kubeclt apply -f local-rate-limit/envoy-filter-local.yaml
```

verify envoy filter binding with nginx pod

```sh
kubectl -n local-rate-limit exec -it <NGINX_POD_NAME> -c istio-proxy -- curl localhost:15000/config_dump > envoyconfig-local.json
```

it should have `envoy.filters.http.local_ratelimit`

```sh
"http_filters": [
  {
    "name": "envoy.filters.http.local_ratelimit",
    "typed_config": {
      "@type": "type.googleapis.com/udpa.type.v1.TypedStruct",
      "type_url": "type.googleapis.com/envoy.extensions.filters.http.local_ratelimit.v3.LocalRateLimit",
      "value": {
        "stat_prefix": "http_local_rate_limiter",
        "token_bucket": {
          "max_tokens": 2,
          "tokens_per_fill": 1,
          "fill_interval": "10s"
        },
        ...
```
test local rate limit by
```sh
kubectl run curl --rm -i --tty --image=curlimages/curl --restart=Never -- \
  curl -i http://nginx-svc.local-rate-limit.svc.cluster.local
```
when run over number of token in timeframe it will return `429 TOO MANY REQUEST`
```sh
HTTP/1.1 429 Too Many Requests
x-local-rate-limit: TOO_MANY_REQUESTS
content-length: 18
content-type: text/plain
date: Sat, 10 May 2025 14:37:32 GMT
server: istio-envoy
x-envoy-decorator-operation: nginx-svc.local-rate-limit.svc.cluster.local:80/*
```

## Global rate limit with source ip
limit at ingress gateway level

```sh
kubectl label ns global-rate-limit istio-injection=enabled
```

```sh
kubeclt apply -f global-rate-limit/container-registry-secrets.yaml
```

```sh
kubeclt apply -f global-rate-limit/nginx.yaml
```

```sh
kubeclt apply -f global-rate-limit/gateway.yaml
```

```sh
INGRESS_GATEWAY_IP=$(kubectl get svc istio-ingressgateway -n istio-system -o jsonpath='{.status.loadBalancer.ingress[0].ip}')

echo $INGRESS_GATEWAY_IP

# In my case I poc by using minikube so I make a turnel to get external IP of svc
minikube tunnel
###
✅  Tunnel successfully started
...
🏃  Starting tunnel for service istio-ingressgateway.
```
```sh
kubeclt apply -f global-rate-limit/redis.yaml
```
```sh
kubeclt apply -f global-rate-limit/configmap.yaml
```
```sh
kubeclt apply -f global-rate-limit/rate-limit-svc.yaml
```
verify rate limit svc
```sh
kubectl port-forward svc/ratelimit 8080:8080 &
curl localhost:8080/json -d '{"domain": "my-ratelimit", "descriptors": [{ "entries": [{ "key": "remote_address", "value": "10.0.0.0"}] }]}'
```
check cluster_name
```sh
istioctl pc cluster -n global-rate-limit deploy/nginx-deployment --fqdn ratelimit.global-rate-limit.svc.cluster.local  --port 8081 -o json | grep serviceName

# output be like
"serviceName": "outbound|8081||ratelimit.global-rate-limit.svc.cluster.local"
```
add it to global-rate-limit/envoy-filter-global.yaml
```sh
...
  rate_limit_service:
    grpc_service:
      envoy_grpc:
        cluster_name: outbound|8081||ratelimit.global-rate-limit.svc.cluster.local
        authority: ratelimit.global-rate-limit.svc.cluster.local
...
```
```sh
kubeclt apply -f global-rate-limit/envoy-filter-global.yaml
```
test by curl ingress gateway ip (I use minikube so I use ngrok for public access)

response of request
```sh
< HTTP/1.1 200 OK
< server: istio-envoy
< date: Sun, 11 May 2025 10:14:44 GMT
< content-type: text/html
< content-length: 615
< last-modified: Wed, 16 Apr 2025 12:01:11 GMT
< etag: "67ff9c07-267"
< accept-ranges: bytes
< x-envoy-upstream-service-time: 1
< x-ratelimit-limit: 5, 5;w=60
< x-ratelimit-remaining: 4
< x-ratelimit-reset: 16
```

when hit limit
```sh
* Request completely sent off
< HTTP/1.1 429 Too Many Requests
< x-envoy-ratelimited: true
< x-ratelimit-limit: 5, 5;w=60
< x-ratelimit-remaining: 0
< x-ratelimit-reset: 28
< date: Sun, 11 May 2025 10:15:32 GMT
< server: istio-envoy
< content-length: 0
```
Ref: https://learncloudnative.com/blog/2023-07-23-global-rate-limiter