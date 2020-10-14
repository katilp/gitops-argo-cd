# Storing secrets with SOPS and KSOPS

A realistic physics analysis workflow consists of several steps, which usually all produce intermediate artifacts that are then consumed by the subsequent steps. We will use [CephFS](https://clouddocs.web.cern.ch/containers/tutorials/cephfs.html) for this purpose. As a last step, we want to copy the workflow result to EOS, which requires a kerberos token for which we will store the keytab as a secret.

## Creating CephFS storage

Let's create a new manila share (size 1 GB), executing the following commands on `lxplus-cloud.cern.ch`:

```shell
# figure out available share types (https://clouddocs.web.cern.ch/file_shares/share_types.html)
manila type-list
manila create --name argo-demo-manila-cephfs-share --share-type "Geneva CephFS Testing" CephFS 1
```

Once created (check via `manila list`), create a user for access:

```shell
manila access-allow argo-demo-manila-cephfs-share cephx argo_storage_user
manila access-list argo-demo-manila-cephfs-share
```

Now we need to create a corresponding storage class as [base/cephfs-sc.yaml](04_secrets/base/cephfs-sc.yaml):

```yaml
apiVersion: storage.k8s.io/v1beta1
kind: StorageClass
metadata:
  name: csi-cephfs-storageclass
provisioner: manila-provisioner
parameters:
  type: "Geneva CephFS Testing"
  zones: nova
  osSecretName: os-trustee
  osSecretNamespace: kube-system
  protocol: CEPHFS
  backend: csi-cephfs
  csi-driver: cephfs.csi.ceph.com
  # id for some_share from `manila list`
  osShareID: <TO_BE_FILLED>
  # id from `manila access-list argo-manila-cephfs-share`
  osShareAccessID: <TO_BE_FILLED>
```

And add a corresponding line to [kustomization.yaml](04_secrets/kustomization.yaml):

```yaml
resources:
...
- base/cephfs-sc.yaml
```

In order to be able to use this volume in an Argo Workflow, we need to store the access credentials as secrets.

## Populating secrets with SOPS

Using [SOPS](https://github.com/mozilla/sops) and [OpenStack Barbican](https://wiki.openstack.org/wiki/Barbican) we can encrypt and decrypt secrets, and therefore store them under version control. This part of the documentation closely follows (and copies where applicable) the [GitOps Getting Started by Ricardo Rocha](https://gitlab.cern.ch/helm/releases/gitops-getting-started#secrets) documentation with a few updates and some changes related to Argo CD.

For the following steps, it is suggested to clone this repository (e.g. `git clone ssh://git@gitlab.cern.ch:7999/clange/gitops-argo-cd.git`) or your fork of it and work directly in it.

### Setup

The default SOPS client does not support Barbican, and therefore a patched version needs to be used. Download the [latest release](https://github.com/clelange/sops/releases) for your system (here Linux, use on `lxplus-cloud`):

```shell
curl -OL https://github.com/clelange/sops/releases/download/v3.6.1-barbican/sops_3.6.1-barbican_Linux_x86_64.tar.gz
tar xzvf sops_3.6.1-barbican_Linux_x86_64.tar.gz
chmod +x sops
mv sops ~/bin/
```

Then make sure you have your OpenStack environment set up.

**Important**: The environment above must match the project owning the Kubernetes cluster where Argo CD is deployed.

If you're relying on kerberos authentication, you'll need to fetch a token first (I did not need it):

```shell
export OS_TOKEN=$(openstack token issue -c id -f value)
```

In case you get an error such as `__init__() got an unexpected keyword argument 'token'`, make sure to `unset OS_TOKEN` and try again.

### Encrypting

SOPS requires a master key that will be used to encrypt the different data encryption keys. Only the master key needs to be in Barbican.
The first step is to generate a master key and store it in barbican:

```shell
export KEY="$(openssl rand -base64 32)\n$(openssl rand -base64 12)"
openstack secret store -s symmetric -p "$(echo -e $KEY)" -n gitops-argo-cd
SECRET_HREF=$(openstack secret list -n gitops-argo-cd -f value | awk '{print $1}')
```

### Creating the keytab secret

Let's create the keytab first (replace `clange` by your own username accordingly):

```shell
ktutil
# Then inside ktutil:
addent -password -p clange@CERN.CH -k 1 -e aes256-cts
addent -password -p clange@CERN.CH -k 1 -e arcfour-hmac-md5
wkt .keytab
q
```

Try if this works:

```shell
kinit -kt .keytab clange
```

Both username and keytab need to be converted using `base64` (again, replace `clange`):

```shell
cat .keytab | base64 -w 0
echo -n 'clange@CERN.CH' | base64
```

Create a new file called `kerberos-keytab-secret.yaml` and fill it with the following content using the output above:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: kerberos-keytab-secret
type: Opaque
data:
  user: <output of echo -n 'clange@CERN.CH' | base64>
  keytab: <output of cat .keytab | base64 -w 0>
```

Do not add this file under version control!

### Creating the encrypted secret

We will create the encrypted secret from the file just created:

```shell
sops -e --barbican "${SECRET_HREF}" --encrypted-regex '^(user)|(.*key.*)$' \
kerberos-keytab-secret.yaml > kerberos-keytab-secret.enc.yaml
```

To decrypt, run:

```shell
sops -d kerberos-keytab-secret.enc.yaml
```

## Using encrypted secrets with Argo CD

The next step is to add the `kerberos-keytab-secret.enc.yaml` file under version control.
Argo CD itself is un-opinionated on [secret management tools](https://argoproj.github.io/argo-cd/operator-manual/secret-management/), and we are going to use [KSOPS](https://github.com/viaduct-ai/kustomize-sops#argo-cd-integration-), but in a [patched version](https://github.com/clelange/kustomize-sops), which supports SOPS with Barbican.

For this to work, we need to extend the Argo CD (attention, not the Workflows one!) [kustomization.yaml](04_secrets-argo-cd/kustomization.yaml) in a new path [4_secrets-argo-cd](4_secrets-argo-cd) to look as follows:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- https://github.com/argoproj/argo-cd/manifests/cluster-install/?ref=stable

patchesStrategicMerge:
- overlays/production/argocd-cm-kustomize-alpha-plugins-patch.yaml
- overlays/production/argocd-repo-server-ksops-patch.yaml
```

The patch files to be added look like this [overlays/production/argocd-cm-kustomize-alpha-plugins-patch.yaml](04_secrets-argo-cd/overlays/production/argocd-cm-kustomize-alpha-plugins-patch.yaml):

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-cm
  labels:
    app.kubernetes.io/name: argocd-cm
    app.kubernetes.io/part-of: argocd
data:
  kustomize.buildOptions: "--enable_alpha_plugins"
```

and [overlays/production/argocd-repo-server-ksops-patch.yaml](04_secrets-argo-cd/overlays/production/argocd-repo-server-ksops-patch.yaml):

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: argocd-repo-server
spec:
  template:
    spec:
      # 1. Define an emptyDir volume which will hold the custom binaries
      volumes:
        - name: custom-tools
          emptyDir: {}
        - name: cloud-config
          hostPath:
            path: /etc/kubernetes/
            type: Directory
      # 2. Use an init container to download/copy custom binaries into the emptyDir
      initContainers:
        - name: install-ksops
          # Match Argo CD Go version
          # see https://github.com/argoproj/argo-cd/blob/stable/Dockerfile
          image: clelange/ksops:v2.2.0-barbican
          command: ["/bin/sh", "-c"]
          args:
            - echo "Installing KSOPS...";
              export PKG_NAME=ksops;
              mv ${PKG_NAME}.so /custom-tools/;
              mv $GOPATH/bin/kustomize /custom-tools/;
              echo "Done.";
          volumeMounts:
            - mountPath: /custom-tools
              name: custom-tools
      # 3. Volume mount the custom binary to the bin directory (overriding the existing version)
      containers:
        - name: argocd-repo-server
          volumeMounts:
            - mountPath: /usr/local/bin/kustomize
              name: custom-tools
              subPath: kustomize
              # Verify this matches a XDG_CONFIG_HOME=/.config env variable
            - mountPath: /.config/kustomize/plugin/viaduct.ai/v1/ksops/ksops.so
              name: custom-tools
              subPath: ksops.so
            - mountPath: /etc/kubernetes/
              name: cloud-config
          # 4. Set the XDG_CONFIG_HOME env variable to allow kustomize to detect the plugin
          env:
            - name: XDG_CONFIG_HOME
              value: /.config
            - name: GOPHERCLOUD_CONFIG
              value: /etc/kubernetes/cloud-config
```

Patch the `argo-cd` app path:

```shell
argocd app patch argo-cd --patch='[{"op": "replace", "path": "/spec/source/path", "value": "04_secrets-argo-cd"}]' --type json
argocd app sync argo-cd sync
```

## Putting things together

Having added all these files, we need to point Argo CD to the new directory [04_secrets](04_secrets):

```shell
argocd app patch argo-workflows --patch='[{"op": "replace", "path": "/spec/source/path", "value": "04_secrets"}]' --type json
argocd app sync argo-workflows sync
```
