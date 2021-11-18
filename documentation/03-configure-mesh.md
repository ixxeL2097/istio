# 03 - Configure Mesh
## Sidecar injection

- https://istio.io/latest/docs/setup/additional-setup/sidecar-injection/#automatic-sidecar-injection

In order to take advantage of all of Istio’s features, pods in the mesh must be running an Istio sidecar proxy.

When enabled in a pod’s namespace, **automatic injection** injects the proxy configuration at **pod** creation time using an admission controller.

**Manual injection** directly modifies configuration, like **deployments**, by adding the proxy configuration into it.

### Automatic injection

To inject the sidecar automatically at namespace level, you need to label it:

```
kubectl label namespace argocd istio-injection=enabled --overwrite
```

Once the namespace is labelled, destroy the pods inside the namespace to process the injection, and you should quickly see pods with an additional container running in it:
```console
[root@workstation ~ ]$ k get pods -n argocd
NAME                                             READY   STATUS    RESTARTS   AGE
argocd-redis-67ff8b4d4f-srklq                    1/1     Running   0          20h
argocd-repo-server-cc67c465b-7lhzg               1/1     Running   0          20h
argocd-server-745dc9965c-kz8vp                   1/1     Running   0          20h
argocd-application-controller-5cc99468c7-jk75r   1/1     Running   0          20h

[root@workstation ~ ]$ k delete pods --all -n argocd
pod "argocd-redis-67ff8b4d4f-srklq" deleted
pod "argocd-repo-server-cc67c465b-7lhzg" deleted
pod "argocd-server-745dc9965c-kz8vp" deleted
pod "argocd-application-controller-5cc99468c7-jk75r" deleted

[root@workstation ~ ]$ k get pods -n argocd
NAME                                             READY   STATUS    RESTARTS   AGE
argocd-redis-67ff8b4d4f-bfjct                    2/2     Running   0          26s
argocd-repo-server-cc67c465b-4pbnn               2/2     Running   0          26s
argocd-application-controller-5cc99468c7-gpbz7   2/2     Running   0          26s
argocd-server-745dc9965c-cfj6c                   2/2     Running   0          26s
```

In the above example, you enabled injection at the namespace level. Injection can also be controlled on a per-pod basis, by configuring the `sidecar.istio.io/inject` label on a pod:

<!-- markdownlint-disable MD038 -->
| Resource          | Label                             | Enabled value | Disabled value |
| :---------------- | :-------------------------------- | :------------ | :------------- |
| Namespace         | `istio-injection`                 | `enabled`     | `disabled`     |
| Pod               | `sidecar.istio.io/inject`         | `"true"`      | `"false"`      |
<!-- markdownlint-enable MD038 -->

If you are using **control plane revisions**, revision specific labels are instead used by a matching `istio.io/rev` label. For example, for a revision named `canary`:

<!-- markdownlint-disable MD038 -->
| Resource          | Enabled label                 | Disabled label                        | 
| :---------------- | :---------------------------- | :------------------------------------ | 
| Namespace         | `istio.io/rev=canary`         | `istio-injection=disabled`            | 
| Pod               | `istio.io/rev=canary`         | `sidecar.istio.io/inject="false"`     | 
<!-- markdownlint-enable MD038 -->

If the `istio-injection` label and the `istio.io/rev` label are both present on the same namespace, the `istio-injection` label will take precedence.