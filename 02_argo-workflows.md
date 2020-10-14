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

Install and sync it as follows:

```shell
argocd app create argo-workflows --repo https://gitlab.cern.ch/clange/gitops-argo-cd.git --path 02_argo-workflows --dest-server https://kubernetes.default.svc --dest-namespace argo --sync-option CreateNamespace=true
argocd app sync argo-workflows
```

The additional parameter `--sync-option CreateNamespace=true` is important, otherwise the namespace won't be created and syncing will fail.

## Running a first workflow

Following the [Argo Workflows Quick Start](https://argoproj.github.io/argo/quick-start/) instructions, install the CLI getting the latest [release](https://github.com/argoproj/argo/releases), for Linux and v2.11.3:

```shell
curl -sLO https://github.com/argoproj/argo/releases/download/v2.11.3/argo-linux-amd64.gz
# Unzip
gunzip argo-linux-amd64.gz
# Make binary executable
chmod +x argo-linux-amd64
# Move binary to path
mv ./argo-linux-amd64 ~/bin/argo
# Test installation
argo version
```

Let's try to run a first workflow:

```shell
argo submit -n argo --watch https://raw.githubusercontent.com/argoproj/argo/master/examples/hello-world.yaml
```

The workflow will complete, but then give an error: `failed to save outputs: Failed to establish pod watch: timed out waiting for the condition`. We need to adjust the workflow RBAC role to make this work. Let's do this by adding a kustomization.

---

Continue to [Kustomization](03_kustomization.md).
