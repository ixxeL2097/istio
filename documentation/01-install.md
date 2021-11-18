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