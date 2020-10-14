# gitops-argo-cd

GitOps using Argo CD at CERN. This repository accompanies the CERN Container Service Webinar "[From zero to running physics analysis workflows with Argo CD](https://indico.cern.ch/event/950886/)".

## Overview

This tutorial consists of several parts that should be followed in order:

1. [Installing Argo CD](#installing-argo-cd) (below)
2. [Managing Argo CD with Argo CD](01_argo-cd.md)
3. [Installing Argo Workflows](02_argo-workflows.md)
4. [Applying Kustomizations](03_kustomization.md)
5. [Storing secrets with SOPS and KSOPS](04_secrets.md)
6. [Declarative Apps](05_declarative.md)

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

The next step is to connect to the Argo CD server. The server is protected by a password, which can be obtained as follows:

```shell
kubectl get pods -n argocd -l app.kubernetes.io/name=argocd-server -o name | cut -d'/' -f 2
```

Log in using the CLI with username `admin` and the password obtained above. The `<ARGOCD_SERVER>` will be `localhost:8080` if you are on the same machine (alternatively the hostname). And also set a new password:

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

---

Continue to [Managing Argo CD with Argo CD](01_argo-cd.md).
