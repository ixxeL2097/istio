# 02 - Configure Ingress
## Kubernetes Ingress

- https://istio.io/latest/docs/tasks/traffic-management/ingress/kubernetes-ingress/

A Kubernetes Ingress Resources exposes HTTP and HTTPS routes from outside the cluster to services within the cluster.

The `kubernetes.io/ingress.class` annotation is required on the `Ingress` resource to tell the Istio gateway controller that it should handle it, otherwise it will be ignored.

example :
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    kubernetes.io/ingress.class: istio
  name: grafana-ingress
spec:
  rules:
  - host: grafana-istio.fredcorp.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: prom-grafana
            port:
              number: 80
```

In Kubernetes `1.18`, a new resource, `IngressClass`, was added, replacing the `kubernetes.io/ingress.class` annotation on the `Ingress` resource. If you are using this resource, you will need to set the controller field to istio.io/ingress-controller. For example:


```yaml
apiVersion: networking.k8s.io/v1
kind: IngressClass
metadata:
  name: istio
spec:
  controller: istio.io/ingress-controller
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: grafana-ingress
spec:
  ingressClassName: istio
  rules:
  - host: grafana-istio.fredcorp.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: prom-grafana
            port:
              number: 80
```

```
kubectl apply -f ingressClass-istio.yaml -n istio-system
kubectl apply -f ingress-grafana.yaml -n monitoring
```

`Ingress` supports specifying TLS settings (https://kubernetes.io/docs/concepts/services-networking/ingress/#tls). This is supported by Istio, but the referenced `Secret` must exist in the namespace of the `istio-ingressgateway` deployment (typically `istio-system`). cert-manager (https://istio.io/latest/docs/ops/integrations/certmanager/) can be used to generate these certificates.

N.B: for kubernetes version under 1.19, the apiVersion `networking.k8s.io/v1` is instead `networking.k8s.io/v1beta1`. Moreover, the `Ingress` resource use a slightly different syntax for the `backend` section.

## Istio Ingress Gateways
### HTTP unsecure Gateway termination

- https://istio.io/latest/docs/tasks/traffic-management/ingress/ingress-control/

Along with support for Kubernetes **Ingress**, Istio offers another configuration model, **Istio Gateway**. A `Gateway` provides more extensive customization and flexibility than `Ingress`, and allows Istio features such as monitoring and route rules to be applied to traffic entering the cluster.

An ingress **Gateway** describes a load balancer operating at the edge of the mesh that receives incoming HTTP/TCP connections. It configures exposed ports, protocols, etc. but, unlike **Kubernetes Ingress Resources**, does not include any traffic routing configuration. Traffic routing for ingress traffic is instead configured using Istio routing rules, exactly in the same way as for internal service requests.

Create an Istio Gateway:

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: http-gateway
spec:
  selector:
    istio: ingressgateway # use Istio default gateway implementation
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "*"
```

```
kubectl apply -f gateway-http-istio.yaml -n istio-system
```

Configure routes for traffic entering via the Gateway:

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: http-grafana
spec:
  hosts:
  - "grafana-istio.fredcorp.com"
  gateways:
  - istio-system/http-gateway
  http:
  - match:
    - uri:
        prefix: /
    route:
    - destination:
        port:
          number: 80
        host: prom-grafana
```

```
kubectl apply -f virtualService-grafana.yaml -n monitoring
```

Note that if your `Gateway` is in a different namespace than your `VirtualService`, then you need to make sure you prefix the gateway reference with that namespace:

```yaml
  gateways:
  - <namespace>/http-gateway
```

### Secured TLS Gateway termination

- https://istio.io/latest/docs/tasks/traffic-management/ingress/secure-ingress/

Create a secret from the public and private key of your certificate:
```
kubectl create secret tls fredcorp-wildcard-cert --key="certs/private.key" --cert="certs/cert.crt" -n istio-system
```

Create a new TLS Istio Gateway:
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: https-gateway
spec:
  selector:
    istio: ingressgateway # use Istio default gateway implementation
  servers:
  - port:
      number: 443
      name: https
      protocol: HTTPS
    tls:
      mode: SIMPLE
      credentialName: fredcorp-wildcard-cert # must be the same as secret
    hosts:
    - "*"
```

```
kubectl apply -f gateway-https-istio.yaml -n istio-system
```

Configure routes for traffic entering via the Gateway:
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: https-grafana
spec:
  hosts:
  - "grafana-istio.fredcorp.com"
  gateways:
  - istio-system/https-gateway
  http:
  - match:
    - uri:
        prefix: /
    route:
    - destination:
        port:
          number: 80
        host: prom-grafana
```

```
kubectl apply -f virtualService-https-grafana.yaml -n monitoring
```

You can also use a mixed HTTP/HTTPS `Gateway` with TLS redirection:

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: mixed-tls-redirect-gateway
spec:
  selector:
    istio: ingressgateway # use Istio default gateway implementation
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    tls:
      httpsRedirect: true
    hosts:
    - "*"
  - port:
      number: 443
      name: https
      protocol: HTTPS
    tls:
      mode: SIMPLE
      credentialName: fredcorp-wildcard-cert # must be the same as secret
    hosts:
    - "*"
```

### Ingress Gateway without TLS Termination

If you prefer to use `Gateway` without TLS termination, you can select a `PASSTHROUGH` mode instead of `SIMPLE`:

```yaml
    tls:
      mode: PASSTHROUGH
```

and then the `VirtualService`:

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: https-grafana
spec:
  hosts:
  - grafana-istio.fredcorp.com
  gateways:
  - istio-system/https-gateway
  tls:
  - match:
    - port: 443
      sniHosts:
      - grafana-istio.fredcorp.com
    route:
    - destination:
        host: prom-grafana
        port:
          number: 80
```