# Kustomization

Argo CD supports a wide range of tools with which Kubernetes manifests can be defined. [kustomize](https://kustomize.io) and [helm](https://helm.sh/) charts seem to be the most popular tools in this context. The [helm documentation for Argo CD](https://argoproj.github.io/argo-cd/user-guide/helm/) contains an example for a [helm-guestbook](https://github.com/argoproj/argocd-example-apps/tree/master/helm-guestbook) in case you are interested following this path. In the following, we will, however, focus on `kustomize`.

Adding the required RBAC manifest to our app is done by adding the corresponding manifest file to the resources list in the [app's kustomization.yaml](03_kustomization/kustomization.yaml) (mind: different directory [03_kustomization](03_kustomization) used here):

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- https://github.com/argoproj/argo/manifests/cluster-install/?ref=stable
- base/workflow-rbac.yaml
```

where the new file [base/workflow-rbac.yaml](03_kustomization/base/workflow-rbac.yaml) contains the definition of a new workflow `ServiceAccount` as well as a corresponding `Role` and `RoleBinding`.

Let's patch our app to point to the new [03_kustomization](03_kustomization) path and try again:

```shell
argocd app patch argo-workflows --patch='[{"op": "replace", "path": "/spec/source/path", "value": "03_kustomization"}]' --type json
```
