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
### Installation (Just Lab)
```
- helm repo add istio https://istio-release.storage.googleapis.com/charts

When installing Istio using Helm, you need to install 3 charts:
istio-base
istiod
istio-ingress

- helm repo update
- helm install istio-base istio/base --namespace istio-system --create-namespace --version 1.26.0 --set profile=demo
- helm install istiod istio/istiod --namespace istio-system --version 1.26.0 --set profile=demo --set pilot.resources.requests.memory=128Mi --set pilot.resources.requests.cpu=250m
- helm install istio-ingress istio/gateway --namespace istio-ingress --create-namespace --version 1.26.0
- kubectl label namespace default istio-injection=enabled

$ kubectl annotate svc istio-ingress -n istio-ingress \
  service.beta.kubernetes.io/aws-load-balancer-type=external \
  service.beta.kubernetes.io/aws-load-balancer-nlb-target-type=ip \
  service.beta.kubernetes.io/aws-load-balancer-scheme=internet-facing \
  --overwrite
service/istio-ingress annotated

$ kubectl rollout restart deployment istio-ingress -n istio-ingress
deployment.apps/istio-ingress restarted
```
### SSL (ACM)
```
$ ls
bye.yaml                 hello.yaml                     private_key_2.txt
certificate_chain_2.txt  hello-bye-gateway.yaml         private_key_decrypted.pem
certificate-2.txt        hello-bye-virtualservice.yaml  sta-api-combined.pem
===
- dos2unix private_key_2.txt
- openssl pkcs8 -in private_key_2.txt -out private_key_decrypted.pem

- OpenSSL will prompt:
Enter pass phrase for private_key_2.txt:
Enter the passphrase from AWS ACM export.

You need to combine the certificate and chain when creating the TLS secret.
Concatenate certificate + chain:
- cat certificate-2.txt certificate_chain_2.txt > sta-api-combined.pem

kubectl create secret tls sta-api-tls-secret \
  --cert=sta-api-combined.pem \
  --key=private_key_decrypted.pem \
  -n istio-ingress

```

# Istio Service Mesh on EKS

This guide provides step-by-step instructions to install and configure Istio service mesh on Amazon EKS.

## Architecture Overview

```
                                    ┌─────────────────────────────────────────────────────────────────┐
                                    │                        AWS EKS Cluster                          │
                                    │                                                                 │
┌──────────────┐                    │  ┌─────────────────────────────────────────────────────────┐    │
│              │                    │  │                  istio-ingress namespace                 │    │
│   Internet   │                    │  │                                                          │    │
│   Client     │                    │  │  ┌─────────────────────────────────────────────────┐    │    │
│              │                    │  │  │              istio-ingress (Pod)                 │    │    │
└──────┬───────┘                    │  │  │                                                  │    │    │
       │                            │  │  │  ┌─────────────────┐  ┌───────────────────────┐  │    │    │
       │  HTTP Request              │  │  │  │   Envoy Proxy   │  │    Gateway Controller │  │    │    │
       │  (port 80)                 │  │  │  │   (Ingress)     │  │                       │  │    │    │
       ▼                            │  │  │  └────────┬────────┘  └───────────────────────┘  │    │    │
                                      │  │  │           │                                     │    │    │
┌──────┴───────┐                    │  │  └───────────┼─────────────────────────────────────┘    │    │
│              │                    │  │              │                                          │    │
│    AWS       │                    │  └──────────────┼──────────────────────────────────────────┘    │
│  Network     │                    │                 │                                               │
│ Load Balancer│────────────────────┼─────────────────┘                                               │
│    (NLB)     │                    │                                                                 │
│              │                    │  ┌──────────────────────────────────────────────────────────┐   │
└──────────────┘                    │  │                     istio-system namespace                │   │
                                    │  │                                                            │   │
                                    │  │  ┌────────────────────────────────────────────────────┐   │   │
                                    │  │  │                    istiod (Pod)                     │   │   │
                                    │  │  │                                                      │   │   │
                                    │  │  │  ┌──────────────┐  ┌───────────────┐  ┌───────────┐  │   │   │
                                    │  │  │  │    Pilot     │  │    Citadel    │  │  Galley   │  │   │   │
                                    │  │  │  │  (Traffic    │  │   (mTLS       │  │ (Config   │  │   │   │
                                    │  │  │  │  Management) │  │   Certificates)│  │ Validation)│  │   │   │
                                    │  │  │  └──────────────┘  └───────────────┘  └───────────┘  │   │   │
                                    │  │  └────────────────────────────────────────────────────┘   │   │
                                    │  │                           │                                 │   │
                                    │  │                           │ XDS API (gRPC)                 │   │
                                    │  │                           │ Config Push                    │   │
                                    │  └───────────────────────────┼─────────────────────────────────┘   │
                                    │                              │                                     │
                                    │  ┌───────────────────────────┼─────────────────────────────────┐   │
                                    │  │                     default namespace                       │   │
                                    │  │                           │                                 │   │
                                    │  │  ┌────────────────────────┼────────────────────────────┐    │   │
                                    │  │  │                   Gateway (CRD)                        │    │   │
                                    │  │  │                                                          │    │   │
                                    │  │  │  selector: istio: ingress                                │    │   │
                                    │  │  │  servers:                                                │    │   │
                                    │  │  │    - port: 80 (HTTP)                                     │    │   │
                                    │  │  │    - hosts: "*"                                          │    │   │
                                    │  │  └────────────────────────────────────────────────────────┘    │   │
                                    │  │                           │                                 │   │
                                    │  │  ┌────────────────────────┼────────────────────────────┐    │   │
                                    │  │  │                VirtualService (CRD)                    │    │   │
                                    │  │  │                                                          │    │   │
                                    │  │  │  hosts: "*"                                               │    │   │
                                    │  │  │  gateways: httpbin-gateway                               │    │   │
                                    │  │  │  route:                                                  │    │   │
                                    │  │  │    destination: httpbin:8000                             │    │   │
                                    │  │  └────────────────────────┼────────────────────────────┘    │   │
                                    │  │                           │                                 │   │
                                    │  │                           ▼                                 │   │
                                    │  │  ┌────────────────────────────────────────────────────┐    │   │
                                    │  │  │                  httpbin (Service)                   │    │   │
                                    │  │  │                                                      │    │   │
                                    │  │  │  ClusterIP: 10.100.x.x                               │    │   │
                                    │  │  │  Port: 8000 → TargetPort: 8080                       │    │   │
                                    │  │  │  Selector: app=httpbin                               │    │   │
                                    │  │  └────────────────────────┬───────────────────────────┘    │   │
                                    │  │                           │                                 │   │
                                    │  │                           ▼                                 │   │
                                    │  │  ┌────────────────────────────────────────────────────┐    │   │
                                    │  │  │              httpbin (Pod)                           │    │   │
                                    │  │  │                                                      │    │   │
                                    │  │  │  ┌────────────────────┐  ┌───────────────────────┐    │    │   │
                                    │  │  │  │   istio-proxy      │  │    httpbin            │    │    │   │
                                    │  │  │  │   (Envoy Sidecar)  │  │    (go-httpbin)       │    │    │   │
                                    │  │  │  │   Port: 15001      │  │    Port: 8080         │    │    │   │
                                    │  │  │  └────────────────────┘  └───────────────────────┘    │    │   │
                                    │  │  │           ▲                         │                  │    │   │
                                    │  │  │           │      mTLS Traffic       │                  │    │   │
                                    │  │  │           └─────────────────────────┘                  │    │   │
                                    │  │  └────────────────────────────────────────────────────┘    │   │
                                    │  └──────────────────────────────────────────────────────────┘   │
                                    │                                                                 │
                                    └─────────────────────────────────────────────────────────────────┘
```

## Traffic Flow

```
1. Client Request
   │
   ▼
2. AWS Network Load Balancer (NLB)
   │  Forwards traffic to istio-ingress pods
   │
   ▼
3. istio-ingress Gateway Pod (istio-ingress namespace)
   │  Envoy proxy receives traffic
   │  Gateway CRD configures listeners (port 80)
   │  VirtualService CRD defines routing rules
   │
   ▼
4. Service Mesh (mTLS encrypted)
   │  Traffic encrypted between proxies
   │  Istiod manages certificates
   │
   ▼
5. httpbin Pod (default namespace)
   │  istio-proxy (Envoy sidecar) receives traffic
   │  Forwards to httpbin container on port 8080
   │
   ▼
6. Response returned to client
```

## Components Explained

### Control Plane (istio-system namespace)

```
┌─────────────────────────────────────────────────────────────┐
│                         istiod                               │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐  │
│  │    Pilot    │  │   Citadel   │  │      Galley         │  │
│  │             │  │             │  │                     │  │
│  │ • Service   │  │ • mTLS      │  │ • Config            │  │
│  │   Discovery │  │   Certs     │  │   Validation        │  │
│  │ • Traffic   │  │ • Identity  │  │ • Ingress           │  │
│  │   Rules     │  │   Management│  │   Config            │  │
│  └─────────────┘  └─────────────┘  └─────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

| Component | Function |
|-----------|----------|
| **Pilot** | Provides service discovery, traffic management (routing, retries, timeouts) |
| **Citadel** | Issues and manages mTLS certificates for service-to-service communication |
| **Galley** | Validates Istio configuration before applying |

### Data Plane (Sidecar Injection)

```
┌───────────────────────────────────────────────────────┐
│                      Application Pod                    │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │                  istio-proxy                     │   │
│  │                   (Envoy)                        │   │
│  │                                                  │   │
│  │  • Intercepts all inbound/outbound traffic      │   │
│  │  • Enforces mTLS encryption                      │   │
│  │  • Applies traffic rules (routing, retries)      │   │
│  │  • Collects metrics and traces                   │   │
│  └─────────────────────────────────────────────────┘   │
│                         │                               │
│                         ▼                               │
│  ┌─────────────────────────────────────────────────┐   │
│  │              Application Container               │   │
│  │                                                  │   │
│  │  • Your application code                        │   │
│  │  • No changes required                          │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
└───────────────────────────────────────────────────────────┘
```

### Ingress Gateway

```
┌─────────────────────────────────────────────────────────────┐
│                   istio-ingress Gateway                     │
│                                                             │
│  ┌───────────────────────────────────────────────────────┐  │
│  │                    Gateway CRD                         │  │
│  │                                                         │  │
│  │  • Listens on port 80 (HTTP)                           │  │
│  │  • Selects pods with label: istio=ingress              │  │
│  │  • Accepts traffic from any host ("*")                 │  │
│  └───────────────────────────────────────────────────────┘  │
│                           │                                 │
│                           ▼                                 │
│  ┌───────────────────────────────────────────────────────┐  │
│  │                 VirtualService CRD                     │  │
│  │                                                         │  │
│  │  • Matches requests to hosts                           │  │
│  │  • Routes to destination service                       │  │
│  │  • Can define retries, timeouts, splits                │  │
│  └───────────────────────────────────────────────────────┘  │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## mTLS (Mutual TLS) Flow

```
┌─────────────┐                                              ┌─────────────┐
│   Pod A     │                                              │   Pod B     │
│             │                                              │             │
│ ┌─────────┐ │    ┌─────────────────────────────────┐      │ ┌─────────┐ │
│ │ Envoy   │ │    │         mTLS Tunnel              │      │ │ Envoy   │ │
│ │ Proxy   │─┼────│                                 │──────┼─│ Proxy   │ │
│ └─────────┘ │    │  ┌─────────────────────────────┐│      │ └─────────┘ │
│             │    │  │    Encrypted Traffic        ││      │             │
│ ┌─────────┐ │    │  │    (X.509 Certificates)     ││      │ ┌─────────┐ │
│ │  App    │ │    │  │                             ││      │ │  App    │ │
│ │Container│ │    │  │    Pod A Certificate ─────► ││      │ │Container│ │
│ └─────────┘ │    │  │    Verifies Pod B           ││      │ └─────────┘ │
│             │    │  │                             ││      │             │
│             │    │  │    Pod B Certificate ◄───── ││      │             │
│             │    │  │    Verifies Pod A           ││      │             │
│             │    │  └─────────────────────────────┘│      │             │
│             │    │                                 │      │             │
└─────────────┘    └─────────────────────────────────┘      └─────────────┘

                           ▲
                           │
                    ┌──────┴──────┐
                    │   istiod    │
                    │  (Citadel)  │
                    │             │
                    │ Issues certs│
                    │ to all      │
                    │ proxies     │
                    └─────────────┘
```

## Installation Steps

### 1. Add Istio Helm Repository

```powershell
helm repo add istio https://istio-release.storage.googleapis.com/charts
helm repo update
```

### 2. Install istio-base (CRDs)

```powershell
helm install istio-base istio/base -n istio-system --set defaultRevision=default --create-namespace
```

### 3. Install istiod (Control Plane)

```powershell
helm install istiod istio/istiod -n istio-system --wait
```

### 4. Verify istiod is Running

```powershell
kubectl get pods -n istio-system
```

Expected output:
```
NAME                      READY   STATUS    RESTARTS   AGE
istiod-xxxxxxxxxx-xxxxx   1/1     Running   0          2m
```

### 5. Install istio-ingress (Gateway)

```powershell
kubectl create namespace istio-ingress
helm install istio-ingress istio/gateway -n istio-ingress
```

### 6. Verify Ingress Gateway

```powershell
kubectl get pods -n istio-ingress
kubectl get svc -n istio-ingress
```

### 7. Enable Sidecar Injection

```powershell
kubectl label namespace default istio-injection=enabled --overwrite
```

### 8. Configure AWS Network Load Balancer (Optional)

```powershell
kubectl annotate svc istio-ingress -n istio-ingress service.beta.kubernetes.io/aws-load-balancer-type=external service.beta.kubernetes.io/aws-load-balancer-nlb-target-type=ip service.beta.kubernetes.io/aws-load-balancer-scheme=internet-facing --overwrite
```

## Deploy Sample Application

### Apply the httpbin sample:

```powershell
kubectl apply -f httpbin.yaml
```

### Verify deployment:

```powershell
kubectl get pods
kubectl get gateways.networking.istio.io
kubectl get virtualservice
```

### Get Load Balancer URL:

```powershell
kubectl get svc istio-ingress -n istio-ingress -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'
```

### Test the application:

```powershell
$LB = kubectl get svc istio-ingress -n istio-ingress -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'
curl http://$LB/get
curl http://$LB/headers
curl http://$LB/status/200
```

## File Structure

```
istio/
├── README.md       # This documentation
└── httpbin.yaml    # Sample application with Gateway and VirtualService
```

## httpbin.yaml Contents

The `httpbin.yaml` file contains:

| Resource | Description |
|----------|-------------|
| Service | ClusterIP service on port 8000 |
| Deployment | go-httpbin container on port 8080 |
| Gateway | Istio Gateway for HTTP traffic on port 80 |
| VirtualService | Routes all traffic to httpbin service |

## Troubleshooting

### Check Istio components status:

```powershell
kubectl get pods -n istio-system
kubectl get pods -n istio-ingress
kubectl logs -n istio-system -l app=istiod
kubectl logs -n istio-ingress -l app=istio-ingress
```

### Check Gateway and VirtualService:

```powershell
kubectl get gateways.networking.istio.io -n default
kubectl get virtualservice -n default
```

### Check sidecar injection:

```powershell
kubectl get namespace default --show-labels
kubectl get pods -o jsonpath='{.items[*].spec.containers[*].name}'
```

### Common Issues

1. **"exec format error"** - Architecture mismatch. Use ARM-compatible images for ARM nodes.

2. **"upstream connect error"** - Service port mismatch. Ensure `targetPort` matches container port.

3. **Gateway not found** - Use `kubectl get gateways.networking.istio.io` instead of `kubectl get gateway`.

4. **Pod stuck in Pending** - Check node resources and autoscaler logs.

## Uninstall

### Remove sample application:

```powershell
kubectl delete -f httpbin.yaml
```

### Remove Istio:

```powershell
helm uninstall istio-ingress -n istio-ingress
helm uninstall istiod -n istio-system
helm uninstall istio-base -n istio-system
kubectl delete namespace istio-ingress istio-system
```

### Remove CRDs (optional - will delete all Istio resources):

```powershell
kubectl get crd -oname | grep istio.io | xargs kubectl delete
```

## Useful Commands

```powershell
# View all Istio resources
kubectl get all -n istio-system
kubectl get all -n istio-ingress

# View Istio configuration
kubectl get gateway -A
kubectl get virtualservice -A
kubectl get destinationrule -A

# Check proxy status (requires istioctl)
istioctl proxy-status

# View Envoy configuration
istioctl proxy-config cluster <pod-name>
istioctl proxy-config listener <pod-name>
```

## References

- [Istio Documentation](https://istio.io/latest/docs/)
- [Istio Helm Installation](https://istio.io/latest/docs/setup/install/helm/)
- [Istio Gateway](https://istio.io/latest/docs/setup/additional-setup/gateway/)
- [AWS Load Balancer Controller](https://kubernetes-sigs.github.io/aws-load-balancer-controller/)
