# 01 - Install Istio
## Install Istio Operator with helm

Official documentation :
- https://istio.io/latest/docs/setup/install/operator/

Add the repo :
```
helm repo add stevehipwell https://stevehipwell.github.io/helm-charts/
helm repo update
helm fetch --untar stevehipwell/istio-operator
```

Install istio enabling `ServiceMonitor`for Prometheus instance and labeling the object with correct prom label (`release=prom`) :
```
helm upgrade -i istio -n istio istio-operator/ --set dashboards.enabled=true \
                                               --set serviceMonitor.enabled \
                                               --set serviceMonitor.additionalLabels.release=prom
```

yaml section :
```yaml
serviceMonitor:
  enabled: false
  additionalLabels: {}
  #   myLabel: myValue
  interval: 1m
```

If not created, you need to create `istio-system` namespace.

## Install Istio instance

With the operator installed, you can now create a mesh by deploying an `IstioOperator` resource. To install the Istio demo configuration profile using the operator, run the following command:
```
kubectl apply -f istio-operator.yaml
```
```yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
metadata:
  namespace: istio-system
  name: example-istiocontrolplane
spec:
  profile: demo
```

You should get quickly some pods running :
```console
[root@workstation ~ ]$ kubectl get pods
NAME                                    READY   STATUS    RESTARTS   AGE
istiod-6c86784695-sx6tp                 1/1     Running   0          36s
svclb-istio-ingressgateway-54mjc        5/5     Running   0          18s
istio-ingressgateway-8577c57fb6-mpcdb   1/1     Running   0          19s
istio-egressgateway-756d4db566-788kf    1/1     Running   0          19s
```

```console
[root@workstation ~ ]$ kubectl get svc
NAME                   TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)                                                                      AGE
istiod                 ClusterIP      10.43.113.154   <none>        15010/TCP,15012/TCP,443/TCP,15014/TCP                                        16h
istio-egressgateway    ClusterIP      10.43.71.249    <none>        80/TCP,443/TCP                                                               16h
istio-ingressgateway   LoadBalancer   10.43.179.203   172.19.0.3    15021:32339/TCP,80:32669/TCP,443:32713/TCP,31400:31885/TCP,15443:30514/TCP   16h
```

## Monitoring : Prometheus

While you can build your own dashboards, Istio offers a set of preconfigured dashboards for all of the most important metrics for the mesh and for the control plane.

- **Mesh** Dashboard provides an overview of all services in the mesh.
- **Service** Dashboard provides a detailed breakdown of metrics for a service.
- **Workload** Dashboard provides a detailed breakdown of metrics for a workload.
- **Performance** Dashboard monitors the resource usage of the mesh.
- **Control Plane** Dashboard monitors the health and performance of the control plane.

ID of dashboards are the following : 7639, 11829, 7636, 7630, 7645.

The helm chart operator deploys a `ServiceMonitor` for itself in the namespace where it is installed. But this `ServiceMonitor` is not enough to scrape metrics of the Istio instance.

You can use this `ServiceMonitor` and `PodMonitor` to add information to the grafana dashboard. Just make sure that the `labels.release` section match the needed key word for your prometheus operator.:

- https://github.com/istio/istio/blob/master/samples/addons/extras/prometheus-operator.yaml

another resource :

- https://prune998.medium.com/prometheus-operator-and-istio-telemetry-v2-8be5e073272

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PodMonitor
metadata:
  name: envoy-stats-monitor
  namespace: istio-system
  labels:
    monitoring: istio-proxies
    release: prom
spec:
  selector:
    matchExpressions:
    - {key: istio-prometheus-ignore, operator: DoesNotExist}
  namespaceSelector:
    any: true
  jobLabel: envoy-stats
  podMetricsEndpoints:
  - path: /stats/prometheus
    interval: 15s
    relabelings:
    - action: keep
      sourceLabels: [__meta_kubernetes_pod_container_name]
      regex: "istio-proxy"
    - action: keep
      sourceLabels: [__meta_kubernetes_pod_annotationpresent_prometheus_io_scrape]
    - sourceLabels: [__address__, __meta_kubernetes_pod_annotation_prometheus_io_port]
      action: replace
      regex: ([^:]+)(?::\d+)?;(\d+)
      replacement: $1:$2
      targetLabel: __address__
    - action: labeldrop
      regex: "__meta_kubernetes_pod_label_(.+)"
    - sourceLabels: [__meta_kubernetes_namespace]
      action: replace
      targetLabel: namespace
    - sourceLabels: [__meta_kubernetes_pod_name]
      action: replace
      targetLabel: pod_name
---
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: istio-component-monitor
  namespace: istio-system
  labels:
    monitoring: istio-components
    release: prom
spec:
  jobLabel: istio
  targetLabels: [app]
  selector:
    matchExpressions:
    - {key: istio, operator: In, values: [pilot]}
  namespaceSelector:
    any: true
  endpoints:
  - port: http-monitoring
    interval: 15s
  ```

Or you can use these custom `ServiceMonitor` and `PodMonitor` to add information to the grafana dashboard. 

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PodMonitor
metadata:
  labels:
    release: prom
  name: istio-sidecars
spec:
  selector:
    matchLabels:
      security.istio.io/tlsMode: 'istio'
  namespaceSelector:
    any: true
  podMetricsEndpoints:
    - port: http-envoy-prom
      path: /stats/prometheus
---
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  labels:
    release: prom
  name: istio-ingressgateway
spec:
  selector:
    matchLabels:
      istio: ingressgateway
  namespaceSelector:
    matchNames:
      - istio-system
  endpoints:
    - targetPort: http-envoy-prom
      path: /stats/prometheus
---
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  labels:
    release: prom
  name: istiod
spec:
  selector:
    matchLabels:
      istio: pilot
  namespaceSelector:
    matchNames:
      - istio-system
  endpoints:
    - port: http-monitoring
      interval: 15s
  ```
