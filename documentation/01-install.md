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
istio-egressgateway-756d4db566-tdpmv    1/1     Running   0          2m
istio-ingressgateway-8577c57fb6-kzn8j   1/1     Running   0          2m
istiod-6c86784695-twtlv                 1/1     Running   0          2m13s
svclb-istio-ingressgateway-2dhkn        0/5     Pending   0          2m
svclb-istio-ingressgateway-lk5hr        0/5     Pending   0          2m
```

```console
[root@workstation ~ ]$ kubectl get svc
NAME                   TYPE           CLUSTER-IP     EXTERNAL-IP   PORT(S)                                                                      AGE
istio-egressgateway    ClusterIP      10.43.42.252   <none>        80/TCP,443/TCP                                                               4m25s
istio-ingressgateway   LoadBalancer   10.43.18.156   <pending>     15021:30563/TCP,80:30931/TCP,443:32213/TCP,31400:30024/TCP,15443:31497/TCP   4m25s
istiod                 ClusterIP      10.43.72.123   <none>        15010/TCP,15012/TCP,443/TCP,15014/TCP                                        4m38s
```