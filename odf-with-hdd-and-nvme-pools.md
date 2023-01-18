# OpenShift Data Foundation with two StorageClasses for SSD and HDD PersistentVolumes

## Goal

Use OpenShift Data Foundations (Ceph) to present two highly-available StorageClasses
  * one StorageClass for high-speed SSD-based PersistentVolumes
  * and another StorageClass for slower HDD-based PersistentVolumes

**Please note:** I want two distinct StorageClasses instead of trying to create one StorageClass that does [automatic tiering/caching](https://docs.ceph.com/en/latest/rados/operations/cache-tiering/).


## Assumptions

This document only talks about deploying OpenShift Data Foundation (ODF) and the Local Storage Operator (LSO). I've provided some images that illustrate my hardware and their networks, but creating a 3-node "compact" OpenShift cluster on baremetal and configuring the network interfaces is out of scope. Creating Pods and VirtualMachines to consume the storage is also better described elsewhere.


## Overview

I was able to accomplish this by manually creating the `StorageCluster` resource, two `CephBlockPools`, and two `StorageClasses`. Here's a quick diagram that illustrates the hardware I'm using (3 nodes, each with 12 HDDs and 2 NVME) and the resources I created by hand. Unfortunately I don't see a way to accomplish the same result using the simplified Wizard built into the OpenShift Console.

![Diagram of storage devices (HDD/SSD), servers, and Ceph pools](https://i.imgur.com/LcV2CgJ.png)

The SSDs and HDDs in my servers were made available via two different LocalVolumeSets and have corresponding "local" StorageClasses. Then my StorageCluster declares two storageDeviceSets (the GUI Wizard only creates one storageDeviceSet.) Each storageDeviceSet specifies a different deviceType (nvme or hdd) and their dataPVCTemplate requests storage from the appropriate StorageClass. Additionally, I instruct the StorageCluster to `ignore` the creation/reconciliation of it's default CephBlockPool*. After the StorageCluster is created I added two additional CephBlockPool resources and made sure that each CephBlockPool declared an appropriate deviceClass (nvme or hdd.) Finally I created two StorageClasses that referenced the CephBlockPools. Let's describe the process in more detail.

*The default `CephBlockPool` that gets created has a very simply data placement policy/algorithm (CRUSH map) that allows data to be saved on both HDDs and SSDs. Performance will be inconsistent because some pool data will land on SSDs while most of the data (in my case) will use HDD space. *The default `CephFileSystem` and it's ++metadataPool++ and ++dataPool++ also use any/all OSD by default.*


## Prepare the local disks

> ⚠️ **Please note!** The disks need to be clean! If you try to give dirty disks to the Local Storage Operator and/or OpenShift Data Foundation they will be ignored. A disk is considered dirty if it has a filesystem on it, or a partition table. I've also seen disks be ignored because they have LVM data or LUKS/encryption signatures on them. Disks are also ignored if they have an unexpected Ceph OSD Bluestore cluster_id (likely from a previous installation attempt.)

### Install the Local Storage Operator (LSO)

The first task is to make the unused HDDs and SSDs available as PersistentVolumes via a StorageClass. This is done by [installing the Local Storage Operator](https://access.redhat.com/documentation/en-us/red_hat_openshift_data_foundation/4.11/html-single/deploying_openshift_data_foundation_using_bare_metal_infrastructure/index#installing-local-storage-operator_local-bare-metal). The LSO provides four CustomResources: LocalVolume, LocalVolumeSet, LocalVolumeDiscovery & LocalVolumeDiscoveryResult. Creating `LocalVolume` resources is the old way of specifically identifying which storage devices to use (e.g. /dev/sdc or /dev/disk/by-id/...) The new way is to discover available disks by creating a `LocalVolumeDiscovery`. After the disks are discovered, they can be filtered by size and type (ssd / hdd) and turned into PersistentVolumes by creating `LocalVolumeSets`.

> ℹ️ You can add the ODF label to nodes like this:
> `oc label node MY_NODE_NAME cluster.ocs.openshift.io/openshift-storage=''`

An example `LocalVolumeDiscovery`
```
apiVersion: local.storage.openshift.io/v1alpha1
kind: LocalVolumeDiscovery
metadata:
  name: discover-all-devices
  namespace: openshift-local-storage
spec:
  nodeSelector:
    nodeSelectorTerms:
    - matchExpressions:
      - key: cluster.ocs.openshift.io/openshift-storage
        operator: Exists
  tolerations:
  - effect: NoSchedule
    key: node.ocs.openshift.io/storage
    operator: Equal
    value: "true"
```

Example `LocalVolumeSets` for HDD (Rotational) and SSD (NonRotational). The StorageClasses are created for us by the Local Storage Operator!
```
---
apiVersion: local.storage.openshift.io/v1alpha1
kind: LocalVolumeSet
metadata:
  name: local-hdd
  namespace: openshift-local-storage
spec:
  deviceInclusionSpec:
    deviceMechanicalProperties:
      - Rotational
    deviceTypes:
      - disk
    #minSize: 7Ti    ###optional
  nodeSelector:
    nodeSelectorTerms:
    - matchExpressions:
      - key: cluster.ocs.openshift.io/openshift-storage
        operator: Exists
  storageClassName: local-hdd
  volumeMode: Block
```

```
---
apiVersion: local.storage.openshift.io/v1alpha1
kind: LocalVolumeSet
metadata:
  name: local-nvme
  namespace: openshift-local-storage
spec:
  deviceInclusionSpec:
    deviceMechanicalProperties:
      - NonRotational
    deviceTypes:
      - disk
    minSize: 200Gi   ###I need this because my servers have two 120GB SSD for the Operating System, but my OS is only installed on one of the two, so don't touch the other one.
  nodeSelector:
    nodeSelectorTerms:
    - matchExpressions:
      - key: cluster.ocs.openshift.io/openshift-storage
        operator: Exists
  storageClassName: local-nvme
  volumeMode: Block

```


## Deploy OpenShift Data Foundation (ODF)

> :warning: There are notes in the [Troubleshooting](#Appendix) section that talk about cleaning up disks before attempting to redeploy a StorageCluster

In my example you'll see that I'm using [multiple networks to separate the storage traffic](https://access.redhat.com/documentation/en-us/red_hat_openshift_data_foundation/4.11/html-single/deploying_openshift_data_foundation_using_bare_metal_infrastructure/index#creating-multus-networks_local-bare-metal) from other OpenShift traffic. This is done by putting a `network:` section into the `StorageCluster` as shown below. I setup my networks using the NMstate Operator's NodeNetworkConfigurationPolicy resources and creating NetworkAttachmentDefinitions. I think that's out-of-scope for this document. Here's a diagram of what my networks look like.

![OpenShift network diagram](https://i.imgur.com/BYIoRV4.png)


### Install the ODF Operator and create a StorageCluster

After the local disks have been discovered and exposed via multiple StorageClasses we can move on to [installing the OpenShift Data Foundation Operator](https://access.redhat.com/documentation/en-us/red_hat_openshift_data_foundation/4.11/html-single/deploying_openshift_data_foundation_using_bare_metal_infrastructure/index#installing-openshift-data-foundation-operator-using-the-operator-hub_local-bare-metal) and creating our customized `StorageCluster`. The main differences between my `StorageCluster` and one created by the GUI are:
1. `storageDeviceSets:` The GUI only creates one StorageDeviceSet and doesn't specify a `deviceType:` I create two StorageDeviceSets, set the `count:` equal to the expected number of disks, and provide an appropriate `deviceType:` and `storageClassName:`
2. `managedResources:` I disable the creation of a default `cephBlockPool` by setting `reconcileStrategy: ignore`. This avoids having a pool created that puts application data on both SSD and HDDs.
3. `multiCloudGateway:` Because I disable the default `CephBlockPool` I have to provide an alternate `StorageClass` to be used when requesting storage for the NooBaa Postgres DB.

Here is what my complete `StorageCluster` resources looks like. You can get more information about the options by looking at the [`CustomResourceDefinition`](https://github.com/red-hat-storage/ocs-operator/blob/main/deploy/csv-templates/crds/ocs/ocs.openshift.io_storageclusters.yaml).

```
---
apiVersion: ocs.openshift.io/v1
kind: StorageCluster
metadata:
  name: ocs-storagecluster
  namespace: openshift-storage
spec:
#  encryption:
#    clusterWide: true  ### this will use cryptsetup/LUKS for at-rest encryption
  flexibleScaling: true ### allow scaling by node/host instead of by "rack" or by "zone"
  monDataDirHostPath: /var/lib/rook
  network:
    provider: multus
    selectors:
      public: openshift-storage/ocs-public
      cluster: openshift-storage/ocs-cluster
  storageDeviceSets:
  - name: nvme
    count: 6 
    replica: 1 
    deviceType: nvme
    dataPVCTemplate:
      spec:
        storageClassName: local-nvme 
        accessModes:
        - ReadWriteOnce
        volumeMode: Block
        resources:
          requests:
            storage: 1 
  - name: hdd
    count: 30
    replica: 1
    deviceType: hdd
    dataPVCTemplate:
      spec:
        storageClassName: local-hdd
        accessModes:
        - ReadWriteOnce
        volumeMode: Block
        resources:
          requests:
            storage: 1
  managedResources:
    cephBlockPools:
      reconcileStrategy: ignore  ###don't create the default CephBlockPool nor the RBD StorageClass
#    cephFilesystems:
#      reconcileStrategy: ignore  ###don't create the default CephFilesystem nor the CephFS StorageClass
  multiCloudGateway:
    dbStorageClassName: ocs-storagecluster-ceph-rbd-nvme
```

The Rook operator will be notified when the `StorageCluster` resource is created and begin doing a lot of work. It takes a few minutes for the Rook operator to create a quorum of Ceph Monitors (MONs) and deploy a Ceph Manager (MGR). Then the Rook operator will create a Ceph Object Storage Daemon (OSD) for every storage device. In the example above we requested 6 NVMe and 30 HDD devices so we expect to have 36 OSD pods.
> :warning: See the section below about verifying your OSD count and what to do if you're missing OSD pods.

You can observe the Rook Operator's progress and logs with this command:
`oc logs -f -l app=rook-ceph-operator -n openshift-storage`

The `StorageCluster` will report `Status: Progressing` until we create the `StorageClass` listed in the `multiCloudGateway:` section.

### Create CephBlockPools

After the Rook Operator finishes creating the `StorageCluster` you can create `CephBlockPools` with a `deviceClass:` definition. The `deviceClass:` configuration will make sure the pools use either HDDs or SSDs/NVMEs to store data. And the [`targetSizeRatio:`](https://docs.ceph.com/en/latest/rados/operations/placement-groups/#specifying-expected-pool-size) percentage is used to calculate an inital number of Placement Groups to avoid unnecessarily relocating data when the pool starts to fill up.

**Update**: I just discovered that the OpenShift console allows you to create `CephBlockPools` via the GUI. The screens to create them are revealed when creating a new `StorageClass` and choosing the `openshift-storage.rbd.csi.ceph.com` Provisioner. You can choose an existing Storage Pool, or create a new one as shown below. Unfortunately you can't create a new CephFS Filesystem when choosing the other `...csi.ceph.com` Provisioner. :face_with_raised_eyebrow: 

![](https://i.imgur.com/kKUdOEH.png)


The YAML examples below create Ceph Pools that will keep three copies of application data on three separate servers. Actually the application data will be stored in separate "racks." The deafult is to label each node as if it were in a rack of its own. [Other options](https://docs.ceph.com/en/quincy/rados/operations/crush-map/#crush-location) would be to set the `failureDomain:` to "osd" or "host". This provides your applications and VMs high availability and fault tolerance in the event that any one node goes offline, perhaps while rebooting for maintenance or while applying OpenShift updates. The 

```
---
apiVersion: ceph.rook.io/v1
kind: CephBlockPool
metadata:
  name: ocs-storagecluster-cephblockpool-hdd
  namespace: openshift-storage
spec:
  deviceClass: hdd
  enableRBDStats: true
  failureDomain: rack
  replicated:
    replicasPerFailureDomain: 1
    size: 3
    targetSizeRatio: 0.49


---
apiVersion: ceph.rook.io/v1
kind: CephBlockPool
metadata:
  name: ocs-storagecluster-cephblockpool-nvme
  namespace: openshift-storage
spec:
  deviceClass: nvme
  enableRBDStats: true
  failureDomain: rack
  replicated:
    replicasPerFailureDomain: 1
    size: 3
    targetSizeRatio: 0.49
```


### Create StorageClasses

A `StorageClass` can create `PersistentVolumes` from a specific pool by setting the `parameters.pool:` value. The other values were copied from the default `StorageClass` ODF created when I had previously installed via the GUI.

```
---
allowVolumeExpansion: true
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ocs-storagecluster-ceph-rbd-hdd
  annotations:
    description: HDD - Provides RWO Filesystem volumes, and RWO and RWX Block volumes
#    storageclass.kubernetes.io/is-default-class: "true"
parameters:
  clusterID: openshift-storage
  csi.storage.k8s.io/controller-expand-secret-name: rook-csi-rbd-provisioner
  csi.storage.k8s.io/controller-expand-secret-namespace: openshift-storage
  csi.storage.k8s.io/fstype: ext4
  csi.storage.k8s.io/node-stage-secret-name: rook-csi-rbd-node
  csi.storage.k8s.io/node-stage-secret-namespace: openshift-storage
  csi.storage.k8s.io/provisioner-secret-name: rook-csi-rbd-provisioner
  csi.storage.k8s.io/provisioner-secret-namespace: openshift-storage
  imageFeatures: layering
  imageFormat: "2"
  pool: ocs-storagecluster-cephblockpool-hdd
provisioner: openshift-storage.rbd.csi.ceph.com
reclaimPolicy: Delete
volumeBindingMode: Immediate
allowVolumeExpansion: true


---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ocs-storagecluster-ceph-rbd-nvme
  annotations:
    description: NVMe - Provides RWO Filesystem volumes, and RWO and RWX Block volumes
parameters:
  clusterID: openshift-storage
  csi.storage.k8s.io/controller-expand-secret-name: rook-csi-rbd-provisioner
  csi.storage.k8s.io/controller-expand-secret-namespace: openshift-storage
  csi.storage.k8s.io/fstype: ext4
  csi.storage.k8s.io/node-stage-secret-name: rook-csi-rbd-node
  csi.storage.k8s.io/node-stage-secret-namespace: openshift-storage
  csi.storage.k8s.io/provisioner-secret-name: rook-csi-rbd-provisioner
  csi.storage.k8s.io/provisioner-secret-namespace: openshift-storage
  imageFeatures: layering
  imageFormat: "2"
  pool: ocs-storagecluster-cephblockpool-nvme
provisioner: openshift-storage.rbd.csi.ceph.com
reclaimPolicy: Delete
volumeBindingMode: Immediate
```

## Conclusion

If all went well, you've now successfully deployed and configured the Local Storage Operator and OpenShift Data Foundation so that applications can choose to put their data either on HDDs or on SSDs. If something doesn't seem right, the bits that follow can be used to troubleshoot, cleanup, and try again. There is, however, one hiccup in the plan so far. The `CephBlockPools` were used to create `StorageClasses` that apply to RWO (ReadWriteOnce) requests. In other words, the `StorageClasses` we created only make `PersistentVolumes` that can be attached by one `Pod`. What about RWX (ReadWriteMany) volumes that can be used simultaneously by many `Pods`? RWX volumes are handled by the **CephFS** `StorageClass` that was automatically created. Unfortunately that CephFS will arbitrarily put some data on HDDs and some on SSDs. I'm still working out the simplest way to correct that issue.

## CephFS
**How do we tell CephFS which OSD (nvme or hdd) to use?**

We can set `StorageCluster.spec.managedResources.cephFilesystems.reconcileStrategy: ignore` and then manually create a [`CephFileSystem` resource](https://github.com/red-hat-storage/ocs-operator/blob/main/deploy/csv-templates/crds/rook/ceph.rook.io_cephfilesystems.yaml). The `CustomResourceDefinition` allows us to specify `CephFileSystem.spec.dataPools.[].deviceClass:` as well as `CephFileSystem.spec.metadataPool.deviceClass:`. We would also need to create our own `StorageClass` for this `CephFilesystem`.

```
---
apiVersion: ceph.rook.io/v1
kind: CephFilesystem
metadata:
  name: ocs-storagecluster-cephfilesystem
  namespace: openshift-storage
spec:
  dataPools:
  - deviceClass: hdd
    failureDomain: host
    replicated:
      replicasPerFailureDomain: 1
      size: 3
      targetSizeRatio: 0.49
    statusCheck:
      mirror: {}
  metadataPool:
    deviceClass: nvme
    failureDomain: host
    replicated:
      replicasPerFailureDomain: 1
      size: 3
  metadataServer:
    activeCount: 1
    activeStandby: true
    priorityClassName: openshift-user-critical
    placement: {}  # Should probably add this
    resources: {}  # Should probably also add this
```

StorageClass YAML
```
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  annotations:
    description: jcall- CephFS with metadata on NVMe and data on HDD
  name: ocs-storagecluster-cephfs
parameters:
  clusterID: openshift-storage
  csi.storage.k8s.io/controller-expand-secret-name: rook-csi-cephfs-provisioner
  csi.storage.k8s.io/controller-expand-secret-namespace: openshift-storage
  csi.storage.k8s.io/node-stage-secret-name: rook-csi-cephfs-node
  csi.storage.k8s.io/node-stage-secret-namespace: openshift-storage
  csi.storage.k8s.io/provisioner-secret-name: rook-csi-cephfs-provisioner
  csi.storage.k8s.io/provisioner-secret-namespace: openshift-storage
  fsName: ocs-storagecluster-cephfilesystem
allowVolumeExpansion: true
provisioner: openshift-storage.cephfs.csi.ceph.com
reclaimPolicy: Delete
volumeBindingMode: Immediate
```

# Appendix

## Not enough OSDs? Restart the operator!

The Local Storage Operator will tell you how many drives you should have:

```
$ oc get lvset -n openshift-local-storage
NAME         STORAGECLASS   PROVISIONED   AGE
local-hdd    local-hdd      34            27d
local-nvme   local-nvme     6             126d
```

Compare that with the number of OSDs that exist:

```
$ oc rook-ceph -n openshift-storage ceph osd stat
oc rook-ceph -n openshift-storage ceph osd stat40 osds: 40 up (since 29m), 40 in (since 30m); epoch: e185
```

If these do not match up, delete the operator pod which will attempt to recreate any OSDs that did not get created correctly.

```
$ oc delete pods -n openshift-storage -l app=rook-ceph-operator
```

## Delete everything and try again

I found this Cleanup ODF Knowledge Base article helpful
https://access.redhat.com/articles/6525111

If you delete the `StorageCluster` resource, it should also delete any resources it created like `CephBlockPools`. Because we created our own `CephBlockPools`, `CephFilesystems`, and `StorageClasses` we have to delete those ourselves before the `StorageCluster` will finish deleting. Of course you can remove `finalizers:` to force an object to delete, but I don't like doing that.

I find that these two commands, **and a lot of patience**, is usually all I need:
:::danger
You need patience because the HDDs, when released from ODF, are reformatted by the Local Storage Operator with an ext2 filesystem (because running mkfs.ext2 is an easy way to issue DISCARD/TRIM commands. TRIM/DISCARD works great for SSD devices and probably EBS or VMware volumes that can pass along the savings to backend storage. But formatting an 8TB HDD at ionice priority level "idle" takes **a long time -- like 20 minutes**! After `mkfs.ext2` finishes `wipefs` is called, the `PersistentVolumes` are marked as `Available` and then the drives are ready to be reused.
:::
:::success
`oc get storagecluster,cephfilesystem,cephblockpool -o name | xargs oc delete --wait=false`

`oc get storageclass | awk '/ceph.com/ {print $1}' | xargs oc delete storageclass`
:::

Here are some commands that I've used
```
# query uninstall mode
oc describe storagecluster/ocs-storagecluster -n openshift-storage | grep uninstall

# delete snapshots
oc get volumesnapshot --all-namespaces -o yaml | oc delete -f -

# remove monitoring configs - remove monitoring PersistantVolumes
oc delete configmap/cluster-monitoring-config -n openshift-monitoring
oc get pvc -n openshift-monitoring -o yaml | oc delete -f -

# remove registry config - remove registry PV
oc patch config.imageregistry.operator.openshift.io/cluster --type json --patch  '[{ "op": "remove", "path": "/spec/storage" },{ "op": "replace", "path": "/spec/replicas", "value": 1 }]'
oc get pvc -n openshift-image-registry -o yaml | oc delete -f -

# remove logging - remove logging PVs
# https://docs.openshift.com/container-platform/4.11/logging/cluster-logging-uninstall.html
oc delete  ClusterLogging/instance -n openshift-logging
oc get pvc -n openshift-logging -o yaml | oc delete -f -

# remove-is-default-annotation
TODO - oc get storageclass

# remove ODF storageClasses
oc get storageclass | awk '/ocs-storagecluster-ceph*/ {print "oc delete storageclass/" $1}' | /bin/sh


# remove all PersistantVolumeClaims (DANGER!)
oc get pvc -A | awk '/ocs-storagecluster-ceph*/ {print "oc delete -n " $1 " pvc/" $2}' | /bin/sh

# remove ObjectBucketClaims
TODO - oc get obc -A

# remove VirtualMachines (DANGER!)
oc get vm -A -o yaml | oc delete -f -

# remove DataVolumes - created & used by OpenShift Virtualization
oc get datavolume -A -o yaml | oc delete -f -

# remove custom CephBlockPools
oc delete -n openshift-storage cephblockpool --all

# remove "released" PersistentVolumes
oc get pv | awk '/ocs-storagecluster-ceph*/ {print "oc delete pv/" $1}' | /bin/sh
```

## Clean the disks (wipe, zap, shred, etc...)

:::warning
The Local Storage Operator will cleanup any `PersistentVolumes` (HDDs, SSDs, NVMEs...) that are `Released` when their `PersistentVolumeClaim` is deleted. This happens when you delete the `StorageCluster`. The `Reclaim` process executes a `quick_reset.sh` script that takes A LONG TIME to complete when working with real HDDs. My 8TB HDDs can take up to **20 minutes** to execute the script! The script first creates an ext2 filesystem as a way to to TRIM/DISCARD data and then uses wipefs to remove the ext2 filesystem. If you see errors about "detected ext2" when recreating your OSD pods, it usually means you didn't let the `quick_reset.sh` script finish.
:::

The Ceph OSD Bluestore signature is difficult to detect, so we just use `sgdisk` to "zap"/erase the partition table regions
https://rook.github.io/docs/rook/v1.3/ceph-teardown.html#zapping-devices

The example above, as well as a GUI deployment, will store Ceph Monitor (MON) data in a `HostPath` on the OpenShift Node instead of in a `PersistentVolume`. We need to check each node and make sure the `/var/lib/rook` directory gets deleted before trying to reinstall.

```
for i in $(oc get node -l cluster.ocs.openshift.io/openshift-storage= -o name); do
  oc debug $i -- chroot /host ls -l /var/lib/rook 2>/dev/null
  #oc debug $i -- chroot /host rm -rf /var/lib/rook 2>/dev/null
done
```

If you enabled clusterWide encryption it's likely that the LUKS/crypt devices didn't get closed properly. I haven't been successfully cleaning those up via an `oc debug node/...` session, and have had to use SSH instead, like this:

```
ssh -i ~/.ssh/id_rsa_openshift core@node-1.example.com
[core@node-1 ~]$ sudo -i
[root@node-1 ~]# for i in $(dmsetup ls | awk '/block-dmcrypt/ {print $1}'); do
                   cryptsetup luksClose --debug --verbose $i
                 done
[root@node-1 ~]# exit
[core@node-1 ~]$ exit
```

## Watch the logs

ODF has been described as a "meta" Operator. Installing ODF results in four Operators being installed, ODF, OCS, NooBaa, and Rook. I find the most useful information from the logs of the Rook operator. Here are commands to follow the logs of the Operators as they do their work.
```
oc logs -f -l app=rook-ceph-operator
oc logs -f -l noobaa-operator=deployment
oc logs -f -l name=ocs-operator 
oc logs -f -l app.kubernetes.io/name=odf-operator
```

## How to verify data placement policies ([CRUSH maps](https://docs.ceph.com/en/quincy/rados/operations/crush-map/))

> :warning: The `ceph` and `rbd` CLI utilities are not easily accessed in order to prevent accidents from happening.

You can run `ceph` and `rbd` commands from the Rook Operator pod or by installing the [`rook-ceph` Krew plugin](https://github.com/rook/kubectl-rook-ceph). My examples below use the `rook-ceph` Krew plugin.

You can access the Rook Operator pod, and set the required environment variable like this.
```
$ oc exec -it -n openshift-storage deployment/rook-ceph-operator -- /bin/bash
bash-4.4$ export CEPH_CONF=/var/lib/rook/openshift-storage/openshift-storage.config
bash-4.4$ ceph -s
  cluster:
    id:     da426786-2431-41f5-a592-f77bc8a6a959
    health: HEALTH_OK
  
  ...<output trimmed>...
```

Use the `ceph osd pool ls` command to verify your pool was created, and has the appropriate number of replicas, etc...

```
oc rook-ceph -n openshift-storage ceph osd pool ls
oc rook-ceph -n openshift-storage ceph osd pool ls detail
```

Use the `ceph osd crush rule ls` command to list the data placement policies (CRUSH maps) and the `ceph osd crush rule dump example-pool-name` to verify which class of OSDs (hdd, ssd, nvme) will be used to store data.

```
oc rook-ceph -n openshift-storage ceph osd crush rule ls
oc rook-ceph -n openshift-storage ceph osd crush rule dump example-pool-name
```

## Acknowledgments
I am very grateful to the authors of these sources. I learned from their words and adapted their examples to fit my needs. Thank you very much!
* https://ilovett.github.io/docs/rook/master/advanced-configuration.html
* https://red-hat-storage.github.io/ocs-training/training/ocs4/ocs4-additionalfeatures-segregation.html
* https://access.redhat.com/documentation/en-us/red_hat_openshift_data_foundation/4.11/html-single/red_hat_openshift_data_foundation_architecture/index#cluster_creation
* https://ceph.io/en/news/blog/2017/new-luminous-crush-device-classes/
* https://access.redhat.com/articles/6525111
* https://rook.github.io/docs/rook/v1.3/ceph-teardown.html#zapping-devices
* https://red-hat-storage.github.io/ocs-training/training/ocs4/odf4-install-no-ui.html
* https://github.com/red-hat-storage/ocs-operator/blob/main/deploy/csv-templates/crds/ocs/ocs.openshift.io_storageclusters.yaml
* [Install Red Hat OpenShift Data Foundation ... using command line interface](https://access.redhat.com/articles/5692201)
* Moving MON data from `HostPath` to `PVC` and Ceph metadata (Bluestore: RocksDB+WAL) to dedicated `PVC` - https://access.redhat.com/articles/6903731 