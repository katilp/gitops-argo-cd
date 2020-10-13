# Storing secrets with SOPS and KSOPS

A realistic physics analysis workflow consists of several steps, which usually all produce intermediate artifacts that are then consumed by the subsequent steps. We will use [CephFS](https://clouddocs.web.cern.ch/containers/tutorials/cephfs.html) for this purpose.

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

### Creating the CephFS secret

The secret to access CephFS needs to look as follows:

```yaml
apiVersion: v1
kind: Secret
metadata:
    name: os-trustee
    namespace: kube-system
type: Opaque
data:

```