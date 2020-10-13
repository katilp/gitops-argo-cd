# Kustomization

Argo CD supports a wide range of tools with which Kubernetes manifests can be defined. [kustomize](https://kustomize.io) and [helm](https://helm.sh/) charts seem to be the most popular tools in this context. The [helm documentation for Argo CD](https://argoproj.github.io/argo-cd/user-guide/helm/) contains an example for a [helm-guestbook](https://github.com/argoproj/argocd-example-apps/tree/master/helm-guestbook) in case you are interested following this path. In the following, we will, however, focus on `kustomize`.

## Adding base resources

Adding the required RBAC manifest to our app is done by adding the corresponding manifest file to the resources list in the [app's kustomization.yaml](03_kustomization/kustomization.yaml) (mind: different directory [03_kustomization](03_kustomization) used here):

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- https://github.com/argoproj/argo/manifests/cluster-install/?ref=stable
- base/workflow-rbac.yaml
```

where the new file [base/workflow-rbac.yaml](03_kustomization/base/workflow-rbac.yaml) contains the definition of a new workflow `ServiceAccount` as well as a corresponding `Role` and `RoleBinding`.

## Adding overlays

The workflow RBAC manifest is a completely new resource that we had to add. However, we need to make one more change to get things to work. We need to patch the workflow controller `ConfigMap` to make the workflows use the "argo-workflows" `ServiceAccount` by default, and also adjust the `securityContext`.

The workflow controller `ConfigMap` exists already (see [definition in the argoproj repository](https://github.com/argoproj/argo/blob/master/docs/workflow-controller-configmap.yaml)), which means that we cannot add it as a resource, but instead we need to create a so-called [strategic merge patch](https://kubernetes-sigs.github.io/kustomize/api-reference/kustomization/patchesstrategicmerge/) to the [app's kustomization.yaml](03_kustomization/kustomization.yaml):

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- https://github.com/argoproj/argo/manifests/cluster-install/?ref=stable
- base/workflow-rbac.yaml

patchesStrategicMerge:
- overlays/production/workflow-controller-spec-security-patch.yaml
```

The [workflow-controller-spec-security-patch.yaml](03_kustomization/overlays/production/workflow-controller-spec-security-patch.yaml) looks like this:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: workflow-controller-configmap
data:
  # Default values that will apply to all Workflows from this controller, unless overridden on the Workflow-level
  # See more: docs/default-workflow-specs.yaml
  workflowDefaults: |-
    spec:
      serviceAccountName: argo-workflow
      securityContext:
        seLinuxOptions:
          type: "spc_t"

```

The [overlay](https://kubernetes-sigs.github.io/kustomize/api-reference/glossary/#overlay) file is in an additional `production` directory here for conventional reasons. In case one would like to use a `dev` or `staging` environment as well, one would create additional directories in parallel.

## Applying the changes

Let's patch our app to point to the new [03_kustomization](03_kustomization) path and try again:

```shell
argocd app patch argo-workflows --patch='[{"op": "replace", "path": "/spec/source/path", "value": "03_kustomization"}]' --type json
# Look at the diff in the UI, then:
argocd app sync argo-workflows
```

Now we can try to run again:

```shell
argo submit -n argo --watch https://raw.githubusercontent.com/argoproj/argo/master/examples/hello-world.yaml
```

The workflow should succeed.