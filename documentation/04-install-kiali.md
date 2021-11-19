# 04 - Kiali
## Sidecar injection

- https://github.com/kiali/helm-charts

Fetch the chart:
```
helm repo add kiali https://kiali.org/helm-charts
helm fetch --untar kiali/kiali-operator
```
Install the operator and the instance at the same time:
```
helm install \
    --set cr.create=true \
    --set cr.namespace=istio-system \
    --namespace kiali-operator \
    --create-namespace \
    kiali-operator \
    kiali/kiali-operator
```

Kiali requires Prometheus to generate the topology graph, show metrics, calculate health and for several other features. If Prometheus is missing or Kiali can’t reach it, Kiali won’t work properly.

By default, Kiali assumes that Prometheus is available at the URL of the form `http://prometheus.<istio_namespace_name>:9090`, which is the usual case if you are using the Prometheus Istio add-on. If your Prometheus instance has a different service name or is installed to a different namespace, you must manually provide the endpoint where it is available, like in the following example:

```yaml
apiVersion: kiali.io/v1alpha1
kind: Kiali
metadata:
  labels:
    app: kiali-operator
    app.kubernetes.io/instance: kiali-op
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/name: kiali-operator
    app.kubernetes.io/part-of: kiali-operator
    app.kubernetes.io/version: v1.43.0
    argocd.argoproj.io/instance: kiali-op
    helm.sh/chart: kiali-operator-1.43.0
    version: v1.43.0
  name: kiali
  namespace: istio-system
spec:
  deployment:
    accessible_namespaces:
      - '**'
  external_services:
    prometheus:
      url: 'http://prom-kube-prometheus-stack-prometheus.monitoring:9090/'
```