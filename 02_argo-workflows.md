# Installing Argo Workflows

[Argo Workflows](https://argoproj.github.io/projects/argo) is a container-native workflow engine.
We will be using it later to run a physics analysis workflow, but first we need to get it installed.

As for Argo CD, we will create a directory (here [02_argo-workflows](02_argo-workflows)) in our repository and point Argo CD to it. Argo Workflows also comes with a [kustomize cluster-install manifest](https://github.com/argoproj/argo/tree/master/manifests/cluster-install) that we can use in the [02_argo-workflows/kustomization.yaml](02_argo-workflows/kustomization.yaml):

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- https://github.com/argoproj/argo/manifests/cluster-install/?ref=stable
```

