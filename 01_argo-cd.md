# Managing Argo CD with Argo CD

A good first step would be to manage Argo CD itself using Argo CD so that we can upgrade it later if needed. You can create the Argo CD app using the web interface and also the CLI (ignore the x509 certificate error for now):

```shell
argocd app create argo-cd --repo https://gitlab.cern.ch/clange/gitops-argo-cd.git --path 01_argo-cd --dest-server https://kubernetes.default.svc --dest-namespace argocd
```

This points to a directory in this repository, [01_argo-cd](01_argo-cd), which only contains a single file [kustomization.yaml](01_argo-cd/kustomization.yaml) with the following content:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- https://github.com/argoproj/argo-cd/manifests/cluster-install/?ref=stable
```

A `Kustomization` can be used to customise Kubernetes objects and resources. In the case here, there is no customisation applied, the file only points to a directory [cluster-install](https://github.com/argoproj/argo-cd/blob/stable/manifests/cluster-install/) in the Argo CD GitHub repository that we used for the installation via `kubectl` above selecting the `stable` tag.

_Important_: In `kustomize`, the URL should follow the [hashicorp/go-getter URL format](https://github.com/hashicorp/go-getter#url-format).

The other parameters of the `argocd` command are setting the destination cluster (`https://kubernetes.default.svc`, this indicates the one Argo CD is installed in) and the destination namespace (`argocd`).

Executing the command will take several seconds, and then you should see it in the web interface as `OutOfSync`, but `Healthy` and also via `argocd app list`. Browse the app to see which resources would be created. You will see that several resources exist already (`argocd app diff argo-cd`) since we've installed the app manually at the beginning.

Let's sychronise it, either using the web interface or:

```shell
argocd app sync argo-cd
```

We now have an Argo CD installation managed via Argo CD controlled via git!

---

Continue to [Installing Argo Workflows](02_argo-workflows.md).
