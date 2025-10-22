## Installation
```
curl -L https://istio.io/downloadIstio | sh -
cd istio-<version-number>
export PATH=$PWD/bin:$PATH
istioctl install --set profile=demo -y

```
### VirtualService vs DestinationRule in Istio
- Both VirtualService and DestinationRule work together to control traffic in Istio, but they serve different purposes.
#### VirtualService (Traffic Routing)
- A VirtualService defines how incoming requests are routed to a service.
It specifies hostnames, traffic rules, path rewrites, weighted routing, retries, and fault injection.

🔹 Key Responsibilities:
- Defines routing rules based on path, headers, or other request properties.
- Supports traffic splitting (e.g., 80% to v1, 20% to v2).
- Performs URI rewriting and redirection.
- Attaches to a Gateway for ingress traffic.

🔹 Example VirtualService (Traffic Splitting 80-20)
```
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: myapp
spec:
  hosts:
  - "myapp.example.com"
  gateways:
  - myapp-gateway
  http:
  - match:
    - uri:
        prefix: /myapp
    route:
    - destination:
        host: myapp
        subset: v1
      weight: 80
    - destination:
        host: myapp
        subset: v2
      weight: 20

```

#### DestinationRule (Traffic Policies & Load Balancing)
- A DestinationRule defines traffic policies (e.g., load balancing, timeouts, connection pools) for a service after it has been routed by a VirtualService.

🔹 Key Responsibilities:
- Defines subsets of a service (e.g., v1, v2 based on labels).
- Configures load balancing (round-robin, least connections, etc.).
- Applies connection pooling (limits requests per connection).
- Controls failover & traffic policies.

🔹 Example DestinationRule (Defining v1 & v2 Subsets)
```
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: myapp
spec:
  host: myapp
  subsets:
    - name: v1
      labels:
        version: v1
    - name: v2
      labels:
        version: v2
  trafficPolicy:
    loadBalancer:
      simple: ROUND_ROBIN
```
🔹 Key Differences

| Feature      | VirtualService                | DestinationRule  |
|--------------|-----------------------------|---------|
| Purpose | Routes requests to the correct service	  | Defines traffic policies for routed requests |
| Traffic Splitting	 | ✅ Yes (e.g., 80% to v1, 20% to v2)  | ❌ No |
| Path-based Routing | ✅ Yes (e.g., /api → backend)	  | ❌ No |
| Subsets | ✅ References subsets  | ✅ Defines subsets (v1, v2) |
| Load Balancing | ❌ No	  | ✅ Yes (Round Robin, Least Connections, etc.) |
| Retries & Timeouts	 | ✅ Yes  | ✅ Yes |
| Connection Pooling	 | ❌ No  | ✅ Yes |

🔹 How They Work Together
- DestinationRule defines subsets (e.g., v1, v2).
- VirtualService routes traffic based on path, weight, or headers.
- Traffic flows through Istio’s Envoy proxies based on these rules.

🔹 Conclusion
- Use VirtualService to control request routing (paths, weights, retries).
- Use DestinationRule to define subsets and apply traffic policies.

# KodeKloud
```
export INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="http2")].nodePort}')
export SECURE_INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="https")].nodePort}')
export TCP_INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="tcp")].nodePort}')
export INGRESS_HOST=$(kubectl get po -l istio=ingressgateway -n istio-system -o jsonpath='{.items[0].status.hostIP}')

---

curl -s -I -HHost:httpbin.example.com "http://$INGRESS_HOST:$INGRESS_PORT/headers"

```