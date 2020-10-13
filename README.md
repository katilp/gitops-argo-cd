# gitops-argo-cd

GitOps using Argo CD at CERN. This repository accompanies the CERN Container Service Webinar "[From zero to running physics analysis workflows with Argo CD](https://indico.cern.ch/event/950886/)".

## Pre-requisites

- An up-and-running Kubernetes cluster, see [CERN CloudDocs instructions](https://clouddocs.web.cern.ch/containers/quickstart.html)
- `kubectl` - e.g. on `lxplus-cloud` or see [installation instructions](https://kubernetes.io/docs/tasks/tools/install-kubectl/)
- Recommended: a virtual machine under your control (to port-forward Kubernetes services instead of defining ingress)

It is probably good to [fork](https://gitlab.cern.ch/clange/gitops-argo-cd/-/forks/new) this repository or create your own from scratch.

## Installing Argo CD

The installation of Argo CD is documented on the [Argo CD web page](https://argoproj.github.io/argo-cd/getting_started/):

```shell
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Install the Argo CD CLI ([instructions](https://argoproj.github.io/argo-cd/cli_installation/) either into AFS your onto a machine under your control):

```shell
VERSION=$(curl --silent "https://api.github.com/repos/argoproj/argo-cd/releases/latest" | grep '"tag_name"' | sed -E 's/.*"([^"]+)".*/\1/')
curl -sSL -o ~/bin/argocd https://github.com/argoproj/argo-cd/releases/download/$VERSION/argocd-linux-amd64
chmod +x ~/bin/argocd
```

On a machine under your control, e.g. an OpenStack VM:

```shell
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Log on to Argo CD using the CLI, get the password:

```shell
kubectl get pods -n argocd -l app.kubernetes.io/name=argocd-server -o name | cut -d'/' -f 2
```

Log in with username `admin` and the password obtained above. The `<ARGOCD_SERVER>` will be `localhost:8080` if you are on the same machine (alternatively the hostname). And also set a new password:

```shell
argocd login <ARGOCD_SERVER>
argocd account update-password
```

Mind: if you would like to access the server from another machine in the CERN network, you might have to open port 8080, e.g. on CC7:

```shell
sudo firewall-cmd --add-port=8080/tcp --permanent
sudo firewall-cmd --reload
sudo firewall-cmd --list-all
```

You should now be able to log in with the same credentials as set above by pointing your browser to [`<ARGOCD_SERVER>`](https://localhost:8080).

## Creating your first app

A good first step would be to manage Argo CD itself using Argo CD so that we can upgrade it later if needed. You can create the Argo CD app using the web interface and also the CLI:

```shell
argocd app create argo-cd --repo https://gitlab.cern.ch/clange/gitops-argo-cd.git --path argo-cd-vanilla --dest-server https://kubernetes.default.svc --dest-namespace argo-cd
```

This points to a directory in this repository, [argo-cd-vanilla](argo-cd-vanilla), which only contains a single file [kustomization.yaml](argo-cd-vanilla/kustomization.yaml) with the following content:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- https://github.com/argoproj/argo-cd/manifests/install.yaml?ref=stable
```

A `Kustomization` can be used to customise Kubernetes objects and resources. In the case here, there is no customisation applied, the file only points to the same [install.yaml](https://github.com/argoproj/argo-cd/blob/stable/manifests/install.yaml) in the Argo CD GitHub repository that we used for the installation via `kubectl` above selecting the `stable` tag (_mind the different URLs_).

## Declarative setup

Argo CD installs several Custom Resource Definitions (CRDs), one of them being an `Application`. See the [Argo CD Declarative Setup documentation](https://argoproj.github.io/argo-cd/operator-manual/declarative-setup/) for more details.

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
    path: argo-cd
  destination:
    server: https://kubernetes.default.svc
    namespace: argo-cd
```
