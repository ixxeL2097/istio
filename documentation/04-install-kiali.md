# 04 - Kiali
## Installation

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

You can override these information in the `values.yaml` of the helm chart under `cr.spec` section:
```yaml
cr:
  create: false
  name: kiali
  # If you elect to create a Kiali CR (--set cr.create=true)
  # and the operator is watching all namespaces (--set watchNamespace="")
  # then this is the namespace where the CR will be created (the default will be the operator namespace).
  namespace: ""

  spec:
    deployment:
      accessible_namespaces:
      - '**'
```

Kiali CR :
- https://github.com/kiali/kiali-operator/blob/master/deploy/kiali/kiali_cr.yaml

By default, Kiali use token authentication (for Kubernetes, and OpenShift for OCP), but you can use "anonymous". Options are "anonymous", "token", "openshift", "openid", "header".

```yaml
cr:
  spec:
    auth:
      strategy: anonymous
```

For ArgoCD :

```yaml
cr:
  spec:
    auth: 
      strategy: anonymous
    external_services:
      prometheus:
        url: 'http://prom-kube-prometheus-stack-prometheus.monitoring:9090/'
      grafana:
        enabled: true
        in_cluster_url: "http://prom-grafana.monitoring.svc.cluster.local:80"
        url: "https://grafana.fredcorp.com"
        auth:
          ca_file: ""
          insecure_skip_verify: true
          password: "admin"
          token: ""
          type: basic
          use_kiali_token: false
          username: "admin"
        dashboards:
        - name: "Istio Service Dashboard"
          variables:
            namespace: "var-namespace"
            service: "var-service"
        - name: "Istio Workload Dashboard"
          variables:
            namespace: "var-namespace"
            workload: "var-workload"
        - name: "Istio Mesh Dashboard"
        - name: "Istio Control Plane Dashboard"
        - name: "Istio Performance Dashboard"
        - name: "Istio Wasm Extension Dashboard"
```

Prometheus specific settings :
```
# **Prometheus-specific settings:
# auth: authentication settings to connect to Prometheus:
#   ca_file: The certificate authority file to use when accessing Prometheus using https. An empty string means no extra
#       certificate authority file is used. Default is an empty string.
#   insecure_skip_verify: Set true to skip verifying certificate validity when Kiali contacts Prometheus over https.
#   password: Password to be used when making requests to Prometheus, for basic authentication. May refer to a secret - see note above.
#   token: Token / API key to access Prometheus, for token-based authentication. May refer to a secret - see note above.
#   type: The type of authentication to use when contacting the server from the Kiali backend. Use "bearer" to send the
#       token to the Prometheus server. Use "basic" to connect with username and password credentials. Use "none" to not use any authentication.
#       Default is "none"
#   use_kiali_token: When true and if auth.type is "bearer", Kiali Service Account token will be used for the API calls to Prometheus,
#       and auth.token config is ignored then.
#   username: Username to be used when making requests to Prometheus, for basic authentication.
# cache_duration: Prometheus caching duration expressed in seconds
# cache_enabled: Enable/disable Prometheus caching used for Health services
# cache_expiration: Prometheus caching expiration expressed in seconds
# custom_headers: A set of name/value settings that will be passed as headers when requests are sent to Prometheus.
# health_check_url: Used in the Components health feature. This is the url which kiali will ping to determine whether the component is reachable or not. It defaults to `url` when not provided.
# is_core: Used in the Components health feature. When true, the unhealthy scenarios will be raised as errors. Otherwise, they will be raised as a warning.
# thanos_proxy: Used to query through thanos. Kiali use the url parameter to query but with the next configurations.
#  enabled: Used to check this configuration when thanos is used
#  retention_period: Thanos Retention period value expresed in string
#  scrape_interval: Thanos Scrape interval value expresed in string
# url: The URL used to query the Prometheus Server. This URL must be accessible from the Kiali pod.
#      If empty, assumes it is in the istio namespace at the URL "http://prometheus.<istio_namespace>:9090"
#    ---
#    prometheus:
#      auth:
#        ca_file: ""
#        insecure_skip_verify: false
#        password: ""
#        token: ""
#        type: "none"
#        use_kiali_token: false
#        username: ""
#      cache_duration: 10
#      cache_enabled: true
#      cache_expiration: 300
#      custom_headers: {}
#      health_check_url: ""
#      is_core: true
#      thanos_proxy:
#        enabled: false
#        retention_period: "7d"
#        scrape_interval: "30s"
#      url: ""
```

Grafana specific settings for Kiali

```
# **Grafana-specific settings:
# auth: authentication settings to connect to Grafana:
#   ca_file: The certificate authority file to use when accessing Grafana using https. An empty string means no extra
#       certificate authority file is used. Default is an empty string.
#   insecure_skip_verify: Set true to skip verifying certificate validity when Kiali contacts Grafana over https.
#   password: Password to be used when making requests to Grafana, for basic authentication. User only requires viewer permissions. May refer to a secret - see note above.
#   token: Token / API key to access Grafana, for token-based authentication. It only requires viewer permissions. May refer to a secret - see note above.
#   type: The type of authentication to use when contacting the server from the Kiali backend. Use "bearer" to send the
#       token to the Grafana server. Use "basic" to connect with username and password credentials. Use "none" to not use any authentication.
#       Default is "none"
#   use_kiali_token: When true and if auth.type is "bearer", the same OAuth token used for authentication in Kiali will be used for the API calls to Grafana,
#       and auth.token config is ignored then.
#   username: Username to be used when making requests to Grafana, for basic authentication. User only requires viewer permissions.
# dashboards: A list of Grafana dashboards that Kiali can link to. Each item contains:
#   name: The name of the dashboard in Grafana
#   variables:
#     app: The name of a variable that holds the app name, if used in that dashboard (else it must be omitted)
#     namespace: The name of a variable that holds the namespace, if used in that dashboard (else it must be omitted)
#     service: The name of a variable that holds the service name, if used in that dashboard (else it must be omitted)
#     workload: The name of a variable that holds the workload name, if used in that dashboard (else it must be omitted)
# enabled: When true, Grafana support will be enabled in Kiali.
# health_check_url: Used in the Components health feature. This is the url which kiali will ping to determine whether the component is reachable or not. It defaults to `in_cluster_url` when not provided.
# in_cluster_url: Set URL for in-cluster access. Example: "http://grafana.istio-system:3000". This URL can contain query parameters if needed, such as "?orgId=1". If not defined, it will default to "http://grafana.<istio_namespace>:3000".
# is_core: Used in the Components health feature. When true, the unhealthy scenarios will be raised as errors. Otherwise, they will be raised as a warning.
# url: The URL that Kiali uses when integrating with Grafana. This URL must be accessible to clients external to
#      the cluster in order for the integration to work properly. If empty, an attempt to auto-discover it is made.
#      This URL can contain query parameters if needed, such as "?orgId=1".
#    ---
#    grafana:
#      auth:
#        ca_file: ""
#        insecure_skip_verify: false
#        password: ""
#        token: ""
#        type: "none"
#        use_kiali_token: false
#        username: ""
#      dashboards:
#      - name: "Istio Service Dashboard"
#        variables:
#          namespace: "var-namespace"
#          service: "var-service"
#      - name: "Istio Workload Dashboard"
#        variables:
#          namespace: "var-namespace"
#          workload: "var-workload"
#      - name: "Istio Mesh Dashboard"
#      - name: "Istio Control Plane Dashboard"
#      - name: "Istio Performance Dashboard"
#      - name: "Istio Wasm Extension Dashboard"
#      enabled: true
#      health_check_url: ""
#      #in_cluster_url
#      is_core: false
#      url: ""
```