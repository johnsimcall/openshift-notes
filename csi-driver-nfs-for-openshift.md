# Connecting OpenShift to NFS storage with csi-driver-nfs

> This procedure is also [available via the GUI](https://hackmd.io/t34YaR0WS7GzNKIXwko8Zg)

Connecting an NFS server, such as a common Linux server, to OpenShift can be a convenient way to provide storage for multi-pod (ReadWriteMany/RWX) Persistent Volumes. *Fancy NFS servers*, like NetApp or Dell/EMC Isilon, have their own Container Storage Interface (CSI) drivers and shouldn't use the csi-driver-nfs described in this document. Using the appropriate CSI driver for your storage provides additional features like efficient snapshots and clones.

The goal of this document is to describe how to install the [csi-driver-nfs](https://github.com/kubernetes-csi/csi-driver-nfs) provisioner onto OpenShift. The procedure can be performed using the Web Console, but I'll document the Helm/command-line method because it's easier to copy and paste.

The process of configuring a Linux server to export NFS shares is [described in the Appendix below](#RHEL-as-an-NFS-server).

**Please note:** The `ServiceAccounts` created by the official [Helm chart](https://github.com/kubernetes-csi/csi-driver-nfs/blob/master/charts/README.md) require additional permissions in order to overcome OpenShift's default security policy and run the `Pods`.

## Overview

I assume that you already have an NFS server available. If not, follow the instructions in the Appendix in order to make a Linux server provide NFS services. Once an NFS server is available, the remaining steps are to:
1. [Make sure `helm` is installed](#Install-the-Helm-CLI-tool)
2. [Connect `helm` to the csi-driver-nfs chart repository](#Add-the-Helm-repo-and-list-available-charts)
3. [Install the chart with certain overrides](#Install-a-chart-release)
4. [Give additional permissions to the `ServiceAccounts`](#Grant-additional-permissions-to-the-ServiceAccounts)
5. [Create a StorageClass](#Create-a-StorageClass)

## Install the Helm CLI tool

You must have Helm installed on your workstation before proceeding. Alternatively you can use the OpenShift's [Web Terminal Operator](https://www.redhat.com/en/blog/a-deeper-look-at-the-web-terminal-operator-1) to run all of the commands from the OpenShift Web Console. The Web Terminal Operator environment has `helm` and `oc` already installed.

```bash
# Download the helm binary and save it to $HOME/bin/helm
curl -L -o $HOME/bin/helm https://mirror.openshift.com/pub/openshift-v4/clients/helm/latest/helm-linux-amd64

# Allow helm to be executed
chmod a+x $HOME/bin/helm

# Confirm that helm is working by reporting the version
helm version
```

## Add the Helm repo and list available charts

Helm uses the term "chart" when referring to the collection of resources that make up a Kubernetes application. Helm charts are published in repos / repositories. The csi-driver-nfs chart instructs OpenShift to create various resources, such as a `Deployment` that creates a controller `Pod` and a `DaemonSet` that creates a mount/unmount `Pod` on each server in the cluster. Two `ServiceAccounts` are also created in order to run the `Pods` with appropriate permissions. 

:::info
These notes were created using version csi-driver-nfs version `4.6.0` on April 2024.
:::

```bash
# Tell helm where to find the csi-driver-nfs chart repository
helm repo add csi-driver-nfs https://raw.githubusercontent.com/kubernetes-csi/csi-driver-nfs/master/charts

# Ask helm to list the repo contents. You should see v4.5.0 and several other versions.
helm search repo -l csi-driver-nfs
```

## Install a chart release

Helm uses the term "release" when referring to an instance of a chart/application running in a Kubernetes cluster. A single chart can be installed many times into the same cluster. Each time its installed, a new release is created. **The csi-driver-nfs chart should only be installed once!** More info available at https://helm.sh/docs/intro/using_helm

My example command below uses several `--set name=value` arguments to override certain default chart values. By overriding the default values I can:
1. Create a new namespace/project for the resources created by csi-driver-nfs
2. Allow the pods that create NFS subdirectories to run on the ControlPlane nodes
3. Have two NFS controller Pods running for high-availability
4. Deploy the optional CSI external snapshot controller
5. Control whether or not the Custom Resource Definitions (CRD) required for the external snapshot controller are installed (they can only be installed once)

:::info
OpenShift installs the external snapshot CRDs by default. Helm will fail and abort when it finds the existing CRDs because the CRDs weren't created (and labeled) by Helm. Its OK to disable the snapshot CRD creation when you see this error:
> Error: INSTALLATION FAILED: rendered manifests contain a resource that already exists. Unable to continue with install: CustomResourceDefinition "volumesnapshots.snapshot.storage.k8s.io" in namespace "" exists and cannot be imported into the current release: invalid ownership metadata; label validation error: missing key "app.kubernetes.io/managed-by": must be set to "Helm"; annotation validation error: missing key "meta.helm.sh/release-name": must be set to "csi-driver-nfs"; annotation validation error: missing key "meta.helm.sh/release-namespace": must be set to "csi-driver-nfs"
:::

**Option 1** - Install everything, but don't try to recreate the external snapshot CRDs that OpenShift installs by default

```bash
helm install csi-driver-nfs csi-driver-nfs/csi-driver-nfs --version v4.6.0 \
  --create-namespace \
  --namespace csi-driver-nfs \
  --set controller.runOnControlPlane=true \
  --set controller.replicas=2 \
  --set controller.strategyType=RollingUpdate \
  --set externalSnapshotter.enabled=true \
  --set externalSnapshotter.customResourceDefinitions.enabled=false
```

**Option 2** - Install on a Single Node Openshift (SNO) cluster. When there is only one node in the cluster, we don't need multiple replicas of the controller pod

```bash
helm install csi-driver-nfs csi-driver-nfs/csi-driver-nfs --version v4.6.0 \
  --create-namespace \
  --namespace csi-driver-nfs \
  --set controller.runOnControlPlane=true \
  --set externalSnapshotter.enabled=true \
  --set externalSnapshotter.customResourceDefinitions.enabled=false
```

## Grant additional permissions to the ServiceAccounts

:::danger
These commands allow the Pods to run as root with all permissions. Ideally specific permissions would be granted with all other permissions restricted. I started working on this in the Appendix, but never finished it. 😭
:::

```bash
oc adm policy add-scc-to-user privileged -z csi-nfs-node-sa -n csi-driver-nfs
oc adm policy add-scc-to-user privileged -z csi-nfs-controller-sa -n csi-driver-nfs
```

## Create a StorageClass

Now that the csi-driver-nfs chart has been installed, and allowed to run with eleveated permissions, it's time to create a `StorageClass`. The `StorageClass` we create must reference the csi-driver-nfs provisioner and provides additional parameters which identify which NFS server and exported folder/share to use. An additional `subDir:` paramater is defined which controls the name of the folder that gets dynamically created when a Persistent Volume Claim is created. 

```yaml
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-csi
provisioner: nfs.csi.k8s.io   
parameters:
  server: nfs-server.example.com   ### NFS server's IP/FQDN
  share:  /example-dir             ### NFS server's exported directory
  subDir: ${pvc.metadata.namespace}-${pvc.metadata.name}-${pv.metadata.name}  ### Folder/subdir name template
reclaimPolicy: Delete
volumeBindingMode: Immediate
```

If the NFS server is only v4 add to the StorageClass:

```yaml
mountOptions:
  - nfsvers=4.1
```

:::warning
I find it helpful to have a default StorageClass. It's possible to have multiple default StorageClasses, but the behavior is undefined. The thought of undefined behavior scares me! 😱

Check that no other StorageClass has been marked as the default with
```bash
oc get storageclass
```

You can make this StorageClass the default by adding an annotation after creating/applying the YAML below
```bash
oc annotate storageclass/nfs-csi storageclass.kubernetes.io/is-default-class=true
```
:::

:::success
All done! The StorageClass has been created, and you can see it working by creating a sample PVC with this YAML

```yaml
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test-nfs
  namespace: csi-driver-nfs
spec:
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
  volumeMode: Filesystem
  storageClassName: nfs-csi   ### remove this line to test the default StorageClass
```

The PVC should become `Bound` in a few seconds...

```bash
oc get pvc -n csi-driver-nfs
NAME       STATUS   VOLUME                 CAPACITY   ACCESS MODES   STORAGECLASS   AGE
test-nfs   Bound    pvc-abcdef-12345-...   1Gi        RWX            nfs-csi        5s
```

:::



# Appendix

## Update the Image Registry's configuration to use shared NFS storage

If you have recently installed OpenShift, and didn't provided storage configuration details for the internal Image Registry, you may be interested in updating the Image Registry's configuration to use the new NFS StorageClass.

:::info
This command will reconfigure the internal Image Registry. The updated configuration will:
- Create the default Persistent Volume Claim from the default StorageClass
  - The default PVC requests a 100GB ReadWriteMany/RWX volume
- Make the internal registry highly available by running 2 pods/replicas
- Revert the management state to Managed if it had been marked as Removed


```bash
oc patch config.imageregistry.operator.openshift.io/cluster --type=merge \
  -p '{"spec":{"rolloutStrategy":"RollingUpdate","replicas":2,"managementState":"Managed","storage":{"pvc":{"claim":""}}}}'
```
:::


:::danger
The command below will terminate the Image Registry pods and delete the default Persistant Volume Claim. PVCs that were created manually will not be deleted.

Only use this if you're planning on changing the internal Image Registry's configuration. All data/images in the Image Registry will be lost!

```bash
oc patch config.imageregistry.operator.openshift.io/cluster --type=merge -p '{"spec":{"managementState":"Removed"}}'
```
:::

## RHEL as an NFS server

Here is a an abbreviated set of commands to configure a RHEL8 server as an NFS server.

```bash
# Install the RPM
dnf install nfs-utils

# Create an export directory with adequate permissions and SElinux labels
mkdir -p /exports/openshift
chmod 755 /exports/openshift
chown -R nfsnobody:nfsnobody /exports/openshift
semanage fcontext --add --type nfs_t "/exports/openshift(/.*)?"
restorecon -R -v /exports/openshift

# Add the directory to the list of exports
echo "/exports/openshift *(insecure,no_root_squash,async,rw)" >> /etc/exports

# Make sure the firewall isn't blocking anything
firewall-cmd --add-service nfs \
             --add-service mountd \
             --add-service rpc-bind \
             --permanent
firewall-cmd --reload

# Start the NFS server service (also starts following reboots)
systemctl enable --now nfs-server

# Check everything from the server's perspective
systemctl status nfs-server
exportfs -av
showmount -e localhost
```

You can mount the export from another Linux system for testing purposes.

```bash
mount nfs.example.com:/exports/openshift /mnt
touch /mnt/test-file
mkdir /mnt/test-dir
ls -lR /mnt            #confirm these folders and files are owned by root
rm -rvf /mnt/test-*
umount /mnt
```

## Ubuntu as an NFS server

:::danger
Ubuntu 22.04 LTS ships with an NFS server configuration that rewrites the group list provided by NFS clients (OpenShift.) In order for Ubuntu to function with csi-driver-nfs and OpenShift you must disable the "`manage-gids=y`" portion of the `/etc/nfs.conf` file using these two commands.

```bash
echo -e "[mountd]\nmanage-gids=n" | sudo tee /etc/nfs.conf.d/disable-mountd-manage-gids.conf

sudo systemctl restart nfs-server nfs-idmapd
```

:::

## Logs and debugging

If you're curious and would like to learn more about how this Provisioner works or troubleshoot and debug failures you can get more information from inspecting the namespace events and pod logs with these commands. The logs below show three log groupings. The first group shows the events in the namespace when a PVC is created. The second grouping shows the logs from a PVC being created which results in a subdirectory being created on the NFS server. The third log grouping shows the logs when a Pod attempts to mount and use the storage.

```bash
oc get events -n csi-driver-nfs &
LAST SEEN   TYPE      REASON                  OBJECT                           MESSAGE
7m28s       Normal    ExternalProvisioning    persistentvolumeclaim/test-nfs   waiting for a volume to be created, either by external provisioner "nfs.csi.k8s.io" or manually created by system administrator
7m28s       Normal    Provisioning            persistentvolumeclaim/test-nfs   External provisioner is provisioning volume for claim "csi-driver-nfs/test-nfs"
7m24s       Normal    ProvisioningSucceeded   persistentvolumeclaim/test-nfs   Successfully provisioned volume pvc-5daf545e-1ca0-4285-ba91-64fd67133b00


oc logs -f -l app=csi-nfs-controller -n csi-driver-nfs &

I1114 18:12:23.858858       1 controller.go:1366] provision "csi-driver-nfs/new450" class "nfs-csi": started
I1114 18:12:23.859290       1 event.go:298] Event(v1.ObjectReference{Kind:"PersistentVolumeClaim", Namespace:"csi-driver-nfs", Name:"new450", UID:"70e4a89e-571c-49e3-adf3-3264c6dc553d", APIVersion:"v1", ResourceVersion:"150582625", FieldPath:""}): type: 'Normal' reason: 'Provisioning' External provisioner is provisioning volume for claim "csi-driver-nfs/new450"
I1114 18:12:24.163417       1 controller.go:923] successfully created PV pvc-70e4a89e-571c-49e3-adf3-3264c6dc553d for PVC new450 and csi volume name rhdata6.dota-lab.iad.redhat.com#data/hdd/ocp-vmware#pvc-70e4a89e-571c-49e3-adf3-3264c6dc553d##
I1114 18:12:24.163682       1 controller.go:1449] provision "csi-driver-nfs/new450" class "nfs-csi": volume "pvc-70e4a89e-571c-49e3-adf3-3264c6dc553d" provisioned
I1114 18:12:24.163752       1 controller.go:1462] provision "csi-driver-nfs/new450" class "nfs-csi": succeeded
I1114 18:12:24.178981       1 event.go:298] Event(v1.ObjectReference{Kind:"PersistentVolumeClaim", Namespace:"csi-driver-nfs", Name:"new450", UID:"70e4a89e-571c-49e3-adf3-3264c6dc553d", APIVersion:"v1", ResourceVersion:"150582625", FieldPath:""}): type: 'Normal' reason: 'ProvisioningSucceeded' Successfully provisioned volume pvc-70e4a89e-571c-49e3-adf3-3264c6dc553d


oc logs -f -l app=csi-nfs-node -c nfs &

I1114 18:23:17.540284       1 utils.go:107] GRPC call: /csi.v1.Node/NodePublishVolume
I1114 18:23:17.540327       1 utils.go:108] GRPC request: {"target_path":"/var/lib/kubelet/pods/ec96ea93-a4d9-4e0b-86a4-8768b324242e/volumes/kubernetes.io~csi/pvc-70e4a89e-571c-49e3-adf3-3264c6dc553d/mount","volume_capability":{"AccessType":{"Mount":{}},"access_mode":{"mode":5}},"volume_context":{"csi.storage.k8s.io/pv/name":"pvc-70e4a89e-571c-49e3-adf3-3264c6dc553d","csi.storage.k8s.io/pvc/name":"new450","csi.storage.k8s.io/pvc/namespace":"csi-driver-nfs","server":"rhdata6.dota-lab.iad.redhat.com","share":"/data/hdd/ocp-vmware","storage.kubernetes.io/csiProvisionerIdentity":"1699985041046-4693-nfs.csi.k8s.io","subdir":"pvc-70e4a89e-571c-49e3-adf3-3264c6dc553d"},"volume_id":"rhdata6.dota-lab.iad.redhat.com#data/hdd/ocp-vmware#pvc-70e4a89e-571c-49e3-adf3-3264c6dc553d##"}
I1114 18:23:17.540807       1 nodeserver.go:123] NodePublishVolume: volumeID(rhdata6.dota-lab.iad.redhat.com#data/hdd/ocp-vmware#pvc-70e4a89e-571c-49e3-adf3-3264c6dc553d##) source(rhdata6.dota-lab.iad.redhat.com:/data/hdd/ocp-vmware/pvc-70e4a89e-571c-49e3-adf3-3264c6dc553d) targetPath(/var/lib/kubelet/pods/ec96ea93-a4d9-4e0b-86a4-8768b324242e/volumes/kubernetes.io~csi/pvc-70e4a89e-571c-49e3-adf3-3264c6dc553d/mount) mountflags([])
I1114 18:23:17.541201       1 mount_linux.go:245] Detected OS without systemd
I1114 18:23:17.541229       1 mount_linux.go:220] Mounting cmd (mount) with arguments (-t nfs rhdata6.dota-lab.iad.redhat.com:/data/hdd/ocp-vmware/pvc-70e4a89e-571c-49e3-adf3-3264c6dc553d /var/lib/kubelet/pods/ec96ea93-a4d9-4e0b-86a4-8768b324242e/volumes/kubernetes.io~csi/pvc-70e4a89e-571c-49e3-adf3-3264c6dc553d/mount)
I1114 18:23:19.898894       1 nodeserver.go:140] skip chmod on targetPath(/var/lib/kubelet/pods/ec96ea93-a4d9-4e0b-86a4-8768b324242e/volumes/kubernetes.io~csi/pvc-70e4a89e-571c-49e3-adf3-3264c6dc553d/mount) since mountPermissions is set as 0
I1114 18:23:19.898932       1 nodeserver.go:142] volume(rhdata6.dota-lab.iad.redhat.com#data/hdd/ocp-vmware#pvc-70e4a89e-571c-49e3-adf3-3264c6dc553d##) mount rhdata6.dota-lab.iad.redhat.com:/data/hdd/ocp-vmware/pvc-70e4a89e-571c-49e3-adf3-3264c6dc553d on /var/lib/kubelet/pods/ec96ea93-a4d9-4e0b-86a4-8768b324242e/volumes/kubernetes.io~csi/pvc-70e4a89e-571c-49e3-adf3-3264c6dc553d/mount succeeded
I1114 18:23:19.898945       1 utils.go:114] GRPC response: {}
```

:::spoiler List of TODO items

## TODO: YAML that gets applied when using the Web / Developer Console

```
---
apiVersion: helm.openshift.io/v1beta1
kind: ProjectHelmChartRepository
metadata:
  name: csi-driver-nfs
  namespace: csi-driver-nfs
spec:
  name: NFS CSI driver for Kubernetes
  connectionConfig:
    url: https://raw.githubusercontent.com/kubernetes-csi/csi-driver-nfs/master/charts
```

```
TODO - add ChartRelease? YAML here
```

## TODO: Create a customized SecurityContextConstraint

ADDITIONAL TESTING REQUIRED HERE

```
---
apiVersion: security.openshift.io/v1
kind: SecurityContextConstraints
metadata:
  name: csi-driver-nfs
allowHostDirVolumePlugin: true
allowHostNetwork: true
allowHostPorts: true
allowPrivilegedContainer: true
allowPrivilegeEscalation: true
allowedCapabilities:
  - SYS_ADMIN
runAsUser:
  type: RunAsAny
seLinuxContext:
  type: RunAsAny

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: system:openshift:scc:csi-driver-nfs
rules:
  - apiGroups:
      - security.openshift.io
    resources:
      - securitycontextconstraints
    resourceNames:
      - csi-driver-nfs
    verbs:
      - use

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: system:openshift:scc:csi-driver-nfs
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:openshift:scc:csi-driver-nfs
subjects:
  - kind: ServiceAccount
    name: csi-nfs-node-sa
    namespace: csi-driver-nfs
  - kind: ServiceAccount
    name: csi-nfs-controller-sa
    namespace: csi-driver-nfs
```
:::
