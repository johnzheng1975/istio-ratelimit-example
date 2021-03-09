
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


### Example 1
- For productpage api, allows 1 requests per minute  
- For other api, allows 10 requests per minute
- 
1. Use the following configmap to [configure the reference implementation](https://github.com/envoyproxy/ratelimit#configuration)
    to rate limit requests to the path `/productpage` at 1 req/min and all other requests at 10 req/min.

    {{< text yaml >}}
    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: ratelimit-config
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
              requests_per_unit: 100
    {{< /text >}}

1. Apply an `EnvoyFilter` to the `ingressgateway` to enable global rate limiting using Envoy's global rate limit filter.

    The first patch inserts the
    `envoy.filters.http.ratelimit` [global envoy filter](https://www.envoyproxy.io/docs/envoy/latest/api-v3/extensions/filters/http/ratelimit/v3/rate_limit.proto#envoy-v3-api-msg-extensions-filters-http-ratelimit-v3-ratelimit) filter into the `HTTP_FILTER` chain.
    The `rate_limit_service` field specifies the external rate limit service, `rate_limit_cluster` in this case.

    The second patch defines the `rate_limit_cluster`, which provides the endpoint location of the external rate limit service.

    {{< text bash >}}
    $ kubectl apply -f - <<EOF
    apiVersion: networking.istio.io/v1alpha3
    kind: EnvoyFilter
    metadata:
      name: filter-ratelimit
      namespace: istio-system
    spec:
      workloadSelector:
        # select by label in the same namespace
        labels:
          istio: ingressgateway
      configPatches:
        # The Envoy config you want to modify
        - applyTo: HTTP_FILTER
          match:
            context: GATEWAY
            listener:
              filterChain:
                filter:
                  name: "envoy.http_connection_manager"
                  subFilter:
                    name: "envoy.router"
          patch:
            operation: INSERT_BEFORE
            # Adds the Envoy Rate Limit Filter in HTTP filter chain.
            value:
              name: envoy.filters.http.ratelimit
              typed_config:
                "@type": type.googleapis.com/envoy.extensions.filters.http.ratelimit.v3.RateLimit
                # domain can be anything! Match it to the ratelimter service config
                domain: productpage-ratelimit
                failure_mode_deny: true
                rate_limit_service:
                  grpc_service:
                    envoy_grpc:
                      cluster_name: rate_limit_cluster
                    timeout: 10s
                  transport_api_version: V3
        - applyTo: CLUSTER
          match:
            cluster:
              service: ratelimit.default.svc.cluster.local
          patch:
            operation: ADD
            # Adds the rate limit service cluster for rate limit service defined in step 1.
            value:
              name: rate_limit_cluster
              type: STRICT_DNS
              connect_timeout: 10s
              lb_policy: ROUND_ROBIN
              http2_protocol_options: {}
              load_assignment:
                cluster_name: rate_limit_cluster
                endpoints:
                - lb_endpoints:
                  - endpoint:
                      address:
                         socket_address:
                          address: ratelimit.default.svc.cluster.local
                          port_value: 8081
    EOF
    {{< /text >}}

1. Apply another `EnvoyFilter` to the `ingressgateway` that defines the route configuration on which to rate limit.
    This adds [rate limit actions](https://www.envoyproxy.io/docs/envoy/latest/api-v3/config/route/v3/route_components.proto#envoy-v3-api-msg-config-route-v3-ratelimit)
    for any route from a virtual host named `*.80`.

    {{< text bash >}}
    $ kubectl apply -f - <<EOF
    apiVersion: networking.istio.io/v1alpha3
    kind: EnvoyFilter
    metadata:
      name: filter-ratelimit-svc
      namespace: istio-system
    spec:
      workloadSelector:
        labels:
          istio: ingressgateway
      configPatches:
        - applyTo: VIRTUAL_HOST
          match:
            context: GATEWAY
            routeConfiguration:
              vhost:
                name: "*:80"
                route:
                  action: ANY
          patch:
            operation: MERGE
            # Applies the rate limit rules.
            value:
              rate_limits:
                - actions: # any actions in here
                  - request_headers:
                      header_name: ":path"
                      descriptor_key: "PATH"
    EOF
    {{< /text >}}



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


