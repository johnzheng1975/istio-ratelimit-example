
# Enabling Rate Limits using Envoy

This task shows you how to use Envoy's native rate limiting to dynamically limit the traffic to an Istio
service. In this task, you will try global rate-limit on the `productpage` service through ingress gateway. 


There will be two examples:



`Example 2:`
- For productpage api, 
  - When X-HPBP-Tenant-ID: tenant01, allows 5 times/min
  - When X-HPBP-Tenant-ID: tenant02, allows 8 times/min
  - When X-HPBP-Tenant-ID is other value, allows 3 times/min
- For other api, allows 10 requests per minute

In order to enchence your understanding, will also change some redis changes in this example.

## Before you begin

1. Setup Istio in a Kubernetes cluster by following the instructions in the
   [Installation Guide](https://istio.io/v1.9/docs/setup/getting-started/).

1. Deploy the [Bookinfo](https://istio.io/v1.9//docs/examples/bookinfo/) sample application.

## Rate limits

Envoy supports two kinds of rate limiting: global and local. Global rate
limiting uses a global gRPC rate limiting service to provide rate limiting for the entire mesh.
Local rate limiting is used to limit the rate of requests per service instance.

In this task you will configure Envoy to rate limit traffic to a specific path of a service
using both global rate limits.

## Architecture

Envoy can be used to [set up global rate limits](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/other_features/global_rate_limiting) for your mesh.
Global rate limiting in Envoy uses a gRPC API for requesting quota from a rate limiting service.
A [reference implementation](https://github.com/envoyproxy/ratelimit) of the API, written in Go with a Redis
backend, is used below.

![alt text](./ratelimit-redis.png)

### Example 1
#### Requirments
- For productpage api, allows 1 requests per minute  
- For other api, allows 10 requests per minute

#### Apply files
```
kubectl apply -f ./example01/
```
#### Verify the results
```
$ curl "http://$GATEWAY_URL/productpage"
# First time, return 200.
# Second time, return 429.

$ curl "http://$GATEWAY_URL/api/v1/products"
# First ten time, return 200.
# Once more, return 429.
```
#### View redis to understand more
```
127.0.0.1:6379> KEYS *
1) "productpage-ratelimit_PATH_/productpage_1615455060"
2) "productpage-ratelimit_PATH_/api/v1/products_1615455120"
127.0.0.1:6379> GET "productpage-ratelimit_PATH_/api/v1/products_1615455120"
"11"
```
#### Understand the config files
1. Below are deployment and service for ratelmit, redis.
    ```
    cat ./example01/20-service-ratelimit-redis.yaml
    ```

1. Use the following configmap to [configure the reference implementation](https://github.com/envoyproxy/ratelimit#configuration)
    to rate limit requests to the path `/productpage` at 1 req/min and all other requests at 10 req/min.
    ```
    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: ratelimit-config
      namespace: ratelimit
    data:
      config.yaml: |
        domain: productpage-ratelimit
        descriptors:
          - key: PATH
            value: "/productpage"
            rate_limit:
              unit: minute
              requests_per_unit: 1
          - key: PATH
            rate_limit:
              unit: minute
              requests_per_unit: 10
    ```

1. Apply an `EnvoyFilter` to the `ingressgateway` to enable global rate limiting using Envoy's global rate limit filter. 
    The first patch inserts the
    `envoy.filters.http.ratelimit` [global envoy filter](https://www.envoyproxy.io/docs/envoy/latest/api-v3/extensions/filters/http/ratelimit/v3/rate_limit.proto#envoy-v3-api-msg-extensions-filters-http-ratelimit-v3-ratelimit) filter into the `HTTP_FILTER` chain.
    The `rate_limit_service` field specifies the external rate limit service, `rate_limit_cluster` in this case.
    The second patch defines the `rate_limit_cluster`, which provides the endpoint location of the external rate limit service.
    ```
    cat ./example01/40-envoyfilter-ratelimit.yaml
    ``` 

1. Apply another `EnvoyFilter` to the `ingressgateway` that defines the route configuration on which to rate limit.
    This adds [rate limit actions](https://www.envoyproxy.io/docs/envoy/latest/api-v3/config/route/v3/route_components.proto#envoy-v3-api-msg-config-route-v3-ratelimit)
    for any route from a virtual host named `*.80`.

    ```
    cat ./example01/40-envoyfilter-ratelimit-svc.yaml
    ```

## Verify the results

### Verify global rate limit

Send traffic to the Bookinfo sample. Visit `http://$GATEWAY_URL/productpage` in your web
browser or issue the following command:

{{< text bash >}}
$ curl "http://$GATEWAY_URL/productpage"
{{< /text >}}

{{< tip >}}
`$GATEWAY_URL` is the value set in the [Bookinfo](/docs/examples/bookinfo/) example.
{{< /tip >}}

You will see the first request go through but every following request within a minute will get a 429 response.


