# Istio Traffic Management Tutorial


## Gateway
Documentation: [Istio Gateway](https://istio.io/latest/docs/reference/config/networking/gateway/)
````yaml
apiVersion: networking.istio.io/v1
kind: Gateway
metadata:
  name: my-gateway
  namespace: some-config-namespace
spec:
  selector:
    app: my-gateway-controller

  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - bookinfo.app
    tls:
      httpsRedirect: true # sends 301 redirect for http requests

  - port:
      number: 443
      name: https-443
      protocol: HTTPS
    hosts:
    - bookinfo.app
    tls:
      mode: SIMPLE # enables HTTPS on this port
      serverCertificate: /etc/certs/servercert.pem
      privateKey: /etc/certs/privatekey.pem
````

## VirtualService
Documentation: [Istio VirtualService](https://istio.io/latest/docs/reference/config/networking/virtual-service/)

The VirtualService defines a set of rules to redirect traffic from the gateway to the service. 

So you should define in the spec section:
- `gateways`: the gateways that should be used to route the traffic
- `hosts`: the hosts that should be routed to the destination
- `http`: the HTTP routes to be used for the traffic, 
    - `name`: the name of the route(any name)
    - `match`: The list of URIs to match
    - `route`: the name of the service(Destination) to route to.

````yaml
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: reviews-route
spec:
  gateways:
  - my-gateway
  hosts:
  - bookinfo.app
  http:
  - name: "productpage-v2-routes"
    match:
    - uri:
        exact: "/productpage"
    - uri:
        prefix: "/static/*"
    - uri:
        prefix: "/login"
    - uri:
        prefix: "/logout"
    - uri:
        prefix: "/api/v1/products"
    route:
    - destination:
        host: productpage.prod.svc.cluster.local # <service-name>.<namespace>.svc.cluster.local
        port:
          number: 8080
        subset: v2 # defined in DestinationRule (Optional)
      weight: 100 # the Percentage of traffic to be routed to the destination
    - destination:
      ...
````
### Fault Injection
There is two types of Fault Injection:
- **Abort**: Aborts a percentage of a requests with a specified HTTP status code. if we don't specify the percentage, The envoy proxy will abort all the requests.
- **Delay**: Delays the request with a specified duration.

Example:

**Abort:**
````yaml
# VirtualService yaml
- route:
  - destination:
      host: customers.default.svc.cluster.local
      subset: v1
    fault:
      abort:
        percentage:
          value: 30
        httpStatus: 404
````
=> This will **abort** 30% of the requests to the customers service with a **404 status code**.

**Delay:**
````yaml
# VirtualService yaml
http:
  - route:
      - destination:
          host: customers.default.svc.cluster.local
          subset: v1
    fault:
      delay:
        percentage:
          value: 30
        fixedDelay: 5s
````    
=> The above setting applies a **5-second** delay to **5%** of incoming requests.

### Timeout
It defines a **maximum duration** for requests. If a request takes longer than the specified **timeout period**, it will be **aborted**. This prevents services from waiting indefinitely for responses, which could slow down the application.
```yaml
http:
  timeout: 5s  
  route:
    - destination:
        host: productpage.prod.svc.cluster.local
        port:
          number: 8080
```
- The timeout can be tested by injecting a delay in the consumed service. 
### Retries
Retries allow Istio to automatically retry failed requests. You can configure:
- The number of retry attempts
- The timeout per retry
- What conditions should trigger a retry (e.g., specific HTTP status codes)

Example:
```yaml
http:
  retries:
    attemps: 3
    perTryTimeout: 2s
  route:
    - destination:
        host: productpage.prod.svc.cluster.local
        port:
          number: 8080
```
=> This will send the request up to 3 times, with a timeout of 2 seconds for each attempt.

**Note**: The fault injection will not trigger the retry policies, For example, if we inject an HTTP 500 error, the retry policy configured to retry on http 500 will not be triggered.

## DestinationRule
Documentation: [Istio DestinationRule](https://istio.io/latest/docs/reference/config/networking/destination-rule/)

It Defines how the Traffic should be routed to the destination(Service) including:
- The subsets(version) to be used by the VirtualService.
- Routing Algorithm (Weighted, Least Request, Round Robin, PassThrough).
- Traffic Policies (TLS, Fault Injection, etc.).

````yaml
apiVersion: networking.istio.io/v1
kind: DestinationRule
metadata:
  name: reviews-destination
spec:
  host: reviews.prod.svc.cluster.local
  trafficPolicy: # Defines the Routing Algorithm and Traffic Policies (OPTIONAL)
    tls: # Traffic Policy for TLS
      mode: ISTIO_MUTUAL
  subsets:
    - name: v1 # Subset name
        labels:
          version: v1 # the version is a label defined in the Deployment Pod template spec
    - name: v2
        labels:
          version: v2  
````
- **Note:** The host field is the name of service(Destination) to be routed to, You can use only the service name as shortcut, but this may lead to issues, because the istio will take namespace of the DestinationRule for the service namespace. So it's better to use the full qualified name of the service(Destination) to be routed to(e.g. `<service-name>.<namespace>.svc.cluster.local`).
- The trafficPolicy section is optional, it can de defined as global to and applied to all the subsets, also a subset can have its own trafficPolicy. Example:
````yaml
spec:
  trafficPolicy: # Global traffic policy, it will be applied to all the subsets
    tls:
      mode: ISTIO_MUTUAL
  subsets:
  - name: v1
    labels:
      version: v1
      trafficPolicy: # Subset traffic policy, it will override the global traffic policy for this subset
        tls:
          mode: SIMPLE
````
### Traffic Policies - load balancing
The load balancing policy defines the algorithm of traffic distribution among the subsets. Istio supports the following algorithms:
- `LEAST_REQUEST`: It **spreads** load across endpoints, favoring endpoints with the least outstanding requests. This is generally **safer** and **outperforms** `ROUND_ROBIN` in nearly all cases.
- `ROUND_ROBIN`(Default): This is generally **unsafe** for many scenarios (e.g. when endpoint weighting is used) as it can overburden endpoints. In general, prefer to **use** `LEAST_REQUEST` as a drop-in **replacement** for `ROUND_ROBIN`.(e.g. req1 -> pod1, req2 -> pod2, req3 -> pod3, req4 -> pod1, req5 -> pod2, req6 -> pod3, etc.)
- `PASSTHROUGH`: This option will forward the connection to the **original IP address** requested by the caller **without** doing any form of **load balancing**. This option must be used with care. It is meant for advanced use cases.
- `RANDOM`: The random load balancer selects a random healthy host. The `Random	` load balancer generally **performs better** than `round robin` if **no health checking** policy is configured.

=> The Traffic is routed only to `Healthy subsets`.

Example:
````yaml
spec:
  trafficPolicy:
    loadBalancer:
      simple: LEAST_REQUEST
````
### Traffic Policies - TLS modes
### Traffic Policies - Circuit Breaking and Outlier Detection
Tutorial: [Circuit Breaking and Outlier Detection - video](https://www.youtube.com/watch?v=dTY4sQdQCzs)

Istio Documentation: [Circuit Breaking and Outlier Detection](https://istio.io/latest/docs/tasks/traffic-management/circuit-breaking/)

It allows to avoid system overlaoding and prevent it from breaking down caused by too many requests. By defining : 
- Threshold for the number of requests that can be handled by the destination service. (Circuit Breaker)
- The thresshold of maximum 5xx returned by the destination service. (Outlier Detection)

The `outlier detection` is used to detect and remove **unhealthy endpoints** from the load balancing pool. And the `circuit breaker` is used to **stop** the client server from sending requests to the destination service when the **pending requests exceed the threshold**.
The Envoy proxy of the client server is the responsible of reading the DestinationRule of the destination service and apply the circuit breaker and outlier detection policies.

Example:
````yaml
# DestinationRule yaml
spec:
  host: customers.default.svc.cluster.local
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 1 # maximum nbr of TCP connections
      http:
        http1MaxPendingRequests: 1 
        maxRequestsPerConnection: 1 
    outlierDetection:
      baseEjectionTime: 3m # The time which the Client server will stop sending requests to the destination service
      consecutiveErrors: 1 # The number of consecutive errors before the endpoint is ejected
      interval: 1s # The interval between checking the health of the endpoints
      maxEjectionPercent: 100 # The percentage of endpoints that can be ejected
````


