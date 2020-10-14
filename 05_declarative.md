# Declarative setup

Argo CD installs several Custom Resource Definitions (CRDs), one of them being an `Application`. See the [Argo CD Declarative Setup documentation](https://argoproj.github.io/argo-cd/operator-manual/declarative-setup/) for more details.

Instead of creating apps individually from the command line, one can also create an "app of apps", which is particularly useful for [cluster bootstrapping](https://argoproj.github.io/argo-cd/operator-manual/cluster-bootstrapping/). An example would be the following [argo-cd](apps/argo-cd.yaml) app:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: argo-cd
  namespace: argocd
  finalizers:
  # Delete resources when deleting app
  - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://gitlab.cern.ch/clange/gitops-argo-cd.git
    targetRevision: HEAD
    path: 04_secrets-argo-cd
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
```

Similarly, one can add the [argo-workflows](apps/argo-workflows.yaml) app into the same [apps](apps) directory:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: argo-workflows
  namespace: argocd
  finalizers:
  # Delete resources when deleting app
  - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://gitlab.cern.ch/clange/gitops-argo-cd.git
    targetRevision: HEAD
    path: 04_secrets
  destination:
    server: https://kubernetes.default.svc
    namespace: argo
```

Mind that it is important that the `Application` itself is installed into the `argocd` namespace whereas its `specs` then end up in a (different) destination namespace.

Another important thing to note is the use of `finalizers`:

```yaml
  finalizers:
  - resources-finalizer.argocd.argoproj.io
```

By default, resources won't be deleted when deleting the app. Adding this `finalizer` will change that. In the example here, we want to keep Argo CD running, but it is OK to delete Argo Workflows.

One can install all apps at once like this:

```
argocd app create apps \
    --dest-namespace argocd \
    --dest-server https://kubernetes.default.svc \
    --repo https://gitlab.cern.ch/clange/gitops-argo-cd.git \
    --path apps
argocd app sync apps
```

---

This has been a first demo of Argo CD. For more, please see the [Argo CD documentation](https://argoproj.github.io/argo-cd/).