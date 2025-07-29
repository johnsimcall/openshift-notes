---
title: Installing OpenShift on a single drive

---

# Installing OpenShift on a single drive

I've heard several customers and home-lab enthusiasts ask if they can install OpenShift on a single node with a single NVMe / SSD drive. This **must be done during installation**, before CoreOS expands it's rootfs and consumes all of the unused space. Slower HDDs shouldn't be used because [etcd needs faster storage](https://etcd.io/docs/v3.5/op-guide/performance/).

In other words, in order to restrict the CoreOS root "/" partition from growing to consume the entire drive, just create a new partition **during installation** with a MachineConfig YAML manifest. LVMStorage can use this empty partition after the installation completes to dynamically allocate storage to VMs and Pods/containers.

:::info
Maybe HPP (hostpath-provisioner) is what I should have documented... But that method also encourages a separate partition - https://github.com/kubevirt/hostpath-provisioner

>  WARNING If you select a directory that shares space with your Operating System, you can potentially exhaust the space on that partition and your node will become non-functional. It is recommended you create a separate partition and point the hostpath provisioner there so it will not interfere with your Operating System.
:::


## Instructions

Create a Butane template and render it into a MachineConfig YAML manifest. Include the custom manifest during the installation process.

:::info
The Assisted Installer for OpenShift at [console.redhat.com](https://console.redhat.com/openshift/assisted-installer/clusters/~new) supports uploading custom manifests now. See [section 3.12 step 14](https://access.redhat.com/documentation/en-us/assisted_installer_for_openshift_container_platform/2024/html-single/installing_openshift_container_platform_with_the_assisted_installer/index#setting-the-cluster-details_installing-with-ui) of the Assisted Installer documentation for more details).
:::

The Butane example that I provide below needs to have a few changes made in order to match your server.

1. update the `device: /dev/nvme0n1` value
2. update the `start_mib: 150000` value. This restricts how large the CoreOS root "/" filesystem can grow (in MiB)
3. update the `size_mib: 0` value. This controls how large your LVMStorage partition will be ("0" means use all availble space)

### Example butane template

This example Butane template gets turned into the MachineConfig YAML manifest
```yaml
---
variant: openshift
version: 4.14.0
metadata:
  labels:
    machineconfiguration.openshift.io/role: master  # for Single Node OpenShift (SNO) and 3-node compact clusters
    #machineconfiguration.openshift.io/role: worker  # only for clusters with dedicated workers
  name: 98-create-a-partition-for-lvmstorage
storage:
  disks:
  - device: /dev/nvme0n1 # adjust as necessary
    partitions:
    - label: lvmstorage  # applying a label to the partition allows us to use nice names like /dev/disk/by-partlabel/lvmstorage instead of /dev/nvme0n1p3 
      start_mib: 150000  # let CoreOS use the first 150GB (minimum 25000 MiB, recommend 120000 MiB or more)
      size_mib:  0       # any size, or "0" to use the rest of the disk
```

Convert the Butane template into MachineConfig YAML with:

```bash
butane create-a-partition-for-lvmstorage.bu -o 98-create-a-partition-for-lvmstorage.yaml 
```


### Example MachineConfig YAML manifest
Copy this final YAML into the OpenShift Assisted Installer web page

```yaml
---
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfig
metadata:
  labels:
    machineconfiguration.openshift.io/role: master
    #machineconfiguration.openshift.io/role: worker
  name: 98-create-a-partition-for-lvmstorage
spec:
  config:
    ignition:
      version: 3.4.0
    storage:
      disks:
        - device: /dev/nvme0n1
          partitions:
            - label: lvmstorage
              startMiB: 150000
              sizeMiB: 0
```

## Creating an LVM-based StorageClass

The final step is to install the LVMStorage Operator and create an `LVMCluster` resource which will dynamically allocate space (logical volumes) to your containers / VMs.
LVM is short for Logical Volume Manager.

An example LVMCluster looks like this

```yaml
---
apiVersion: lvm.topolvm.io/v1alpha1
kind: LVMCluster
metadata:
  name: lvmcluster
  namespace: openshift-storage
spec:
  storage:
    deviceClasses:
      - name: lvmstorage
        deviceSelector:
          paths:
            - /dev/disk/by-partlabel/lvmstorage
        thinPoolConfig:
          name: thin-pool
          overprovisionRatio: 10
          sizePercent: 90
```

# Appendix

:::info
More info about CoreOS partitioning at https://docs.openshift.com/container-platform/4.15/installing/installing_bare_metal/installing-bare-metal.html#installation-user-infra-machines-advanced_disk_installing-bare-metal 

More info about Butane syntax at https://coreos.github.io/butane/config-openshift-v4_14/ 
:::

## Looking at the metal image

```bash
# Download the CoreOS image
wget https://mirror.openshift.com/pub/openshift-v4/dependencies/rhcos/latest/rhcos-4.15.0-x86_64-metal.x86_64.raw.gz

# Install tools that allow us to inspect the CoreOS image
sudo dnf --enablerepo=epel -y install  nbd  nbdkit  nbdkit-filter-gzip

# Present the CoreOS image with a gzip filter and via Unix socket
nbdkit --verbose --foreground --unix - --readonly --filter=gzip file rhcos-4.15.0-x86_64-metal.x86_64.raw.gz &

# Attach /dev/nbd0 to the presented image
sudo nbd-client -u /tmp/nbdkitXqu5MI/socket /dev/nbd0
```

After the CoreOS image is attached we can use tools like `lsblk`, `sgdisk`, and `mount` to insepct it further

```
$ sudo sgdisk --print /dev/nbd0
Disk /dev/nbd0: 7518208 sectors, 3.6 GiB
Sector size (logical/physical): 512/512 bytes
Disk identifier (GUID): 00000000-0000-4000-A000-000000000001
Partition table holds up to 128 entries
Main partition table begins at sector 2 and ends at sector 33
First usable sector is 34, last usable sector is 7518174
Partitions will be aligned on 2048-sector boundaries
Total free space is 2014 sectors (1007.0 KiB)

Number  Start (sector)    End (sector)  Size       Code  Name
   1            2048            4095   1024.0 KiB  EF02  BIOS-BOOT
   2            4096          264191   127.0 MiB   EF00  EFI-SYSTEM
   3          264192         1050623   384.0 MiB   8300  boot
   4         1050624         7518174   3.1 GiB     8300  root

```

```
# find . -name '*growfs*' 2>/dev/null 
./ostree/deploy/rhcos/deploy/808fe52200e7ac9b5fea7979234f8a849b01ff56596e0dba6e5c9ac5d10ca75d.0/usr/lib/dracut/modules.d/40ignition-ostree/ignition-ostree-growfs.service
./ostree/deploy/rhcos/deploy/808fe52200e7ac9b5fea7979234f8a849b01ff56596e0dba6e5c9ac5d10ca75d.0/usr/lib/dracut/modules.d/40ignition-ostree/ignition-ostree-growfs.sh
./ostree/deploy/rhcos/deploy/808fe52200e7ac9b5fea7979234f8a849b01ff56596e0dba6e5c9ac5d10ca75d.0/usr/lib/systemd/systemd-growfs
./ostree/deploy/rhcos/deploy/808fe52200e7ac9b5fea7979234f8a849b01ff56596e0dba6e5c9ac5d10ca75d.0/usr/sbin/xfs_growfs

```# Installing OpenShift on a single drive

I've heard several customers and home-lab enthusiasts ask if they can install OpenShift on a single node with a single NVMe / SSD drive. This **must be done during installation**, before CoreOS expands it's rootfs and consumes all of the unused space. Slower HDDs shouldn't be used because [etcd needs faster storage](https://etcd.io/docs/v3.5/op-guide/performance/).

In other words, in order to restrict the CoreOS root "/" partition from growing to consume the entire drive, just create a new partition **during installation** with a MachineConfig YAML manifest. LVMStorage can use this empty partition after the installation completes to dynamically allocate storage to VMs and Pods/containers.

:::info
Maybe HPP (hostpath-provisioner) is what I should have documented... But that method also encourages a separate partition - https://github.com/kubevirt/hostpath-provisioner

>  WARNING If you select a directory that shares space with your Operating System, you can potentially exhaust the space on that partition and your node will become non-functional. It is recommended you create a separate partition and point the hostpath provisioner there so it will not interfere with your Operating System.
:::


## Instructions

Create a Butane template and render it into a MachineConfig YAML manifest. Include the custom manifest during the installation process.

:::info
The Assisted Installer for OpenShift at [console.redhat.com](https://console.redhat.com/openshift/assisted-installer/clusters/~new) supports uploading custom manifests now. See [section 3.12 step 14](https://access.redhat.com/documentation/en-us/assisted_installer_for_openshift_container_platform/2024/html-single/installing_openshift_container_platform_with_the_assisted_installer/index#setting-the-cluster-details_installing-with-ui) of the Assisted Installer documentation for more details).
:::

The Butane example that I provide below needs to have a few changes made in order to match your server.

1. update the `device: /dev/nvme0n1` value
2. update the `start_mib: 150000` value. This restricts how large the CoreOS root "/" filesystem can grow (in MiB)
3. update the `size_mib: 0` value. This controls how large your LVMStorage partition will be ("0" means use all availble space)

### Example butane template

This example Butane template gets turned into the MachineConfig YAML manifest
```yaml
---
variant: openshift
version: 4.14.0
metadata:
  labels:
    machineconfiguration.openshift.io/role: master  # for Single Node OpenShift (SNO) and 3-node compact clusters
    #machineconfiguration.openshift.io/role: worker  # only for clusters with dedicated workers
  name: 98-create-a-partition-for-lvmstorage
storage:
  disks:
  - device: /dev/nvme0n1 # adjust as necessary
    partitions:
    - label: lvmstorage  # applying a label to the partition allows us to use nice names like /dev/disk/by-partlabel/lvmstorage instead of /dev/nvme0n1p3 
      start_mib: 150000  # let CoreOS use the first 150GB (minimum 25000 MiB, recommend 120000 MiB or more)
      size_mib:  0       # any size, or "0" to use the rest of the disk
```

Convert the Butane template into MachineConfig YAML with:

```bash
butane create-a-partition-for-lvmstorage.bu -o 98-create-a-partition-for-lvmstorage.yaml 
```


### Example MachineConfig YAML manifest
Copy this final YAML into the OpenShift Assisted Installer web page

```yaml
---
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfig
metadata:
  labels:
    machineconfiguration.openshift.io/role: master
    #machineconfiguration.openshift.io/role: worker
  name: 98-create-a-partition-for-lvmstorage
spec:
  config:
    ignition:
      version: 3.4.0
    storage:
      disks:
        - device: /dev/nvme0n1
          partitions:
            - label: lvmstorage
              startMiB: 150000
              sizeMiB: 0
```

## Creating an LVM-based StorageClass

The final step is to install the LVMStorage Operator and create an `LVMCluster` resource which will dynamically allocate space (logical volumes) to your containers / VMs.
LVM is short for Logical Volume Manager.

An example LVMCluster looks like this

```yaml
---
apiVersion: lvm.topolvm.io/v1alpha1
kind: LVMCluster
metadata:
  name: lvmcluster
  namespace: openshift-storage
spec:
  storage:
    deviceClasses:
      - name: lvmstorage
        deviceSelector:
          paths:
            - /dev/disk/by-partlabel/lvmstorage
        thinPoolConfig:
          name: thin-pool
          overprovisionRatio: 10
          sizePercent: 90
```

# Appendix

:::info
More info about CoreOS partitioning at https://docs.openshift.com/container-platform/4.15/installing/installing_bare_metal/installing-bare-metal.html#installation-user-infra-machines-advanced_disk_installing-bare-metal 

More info about Butane syntax at https://coreos.github.io/butane/config-openshift-v4_14/ 
:::

## Looking at the metal image

```bash
# Download the CoreOS image
wget https://mirror.openshift.com/pub/openshift-v4/dependencies/rhcos/latest/rhcos-4.15.0-x86_64-metal.x86_64.raw.gz

# Install tools that allow us to inspect the CoreOS image
sudo dnf --enablerepo=epel -y install  nbd  nbdkit  nbdkit-filter-gzip

# Present the CoreOS image with a gzip filter and via Unix socket
nbdkit --verbose --foreground --unix - --readonly --filter=gzip file rhcos-4.15.0-x86_64-metal.x86_64.raw.gz &

# Attach /dev/nbd0 to the presented image
sudo nbd-client -u /tmp/nbdkitXqu5MI/socket /dev/nbd0
```

After the CoreOS image is attached we can use tools like `lsblk`, `sgdisk`, and `mount` to insepct it further

```
$ sudo sgdisk --print /dev/nbd0
Disk /dev/nbd0: 7518208 sectors, 3.6 GiB
Sector size (logical/physical): 512/512 bytes
Disk identifier (GUID): 00000000-0000-4000-A000-000000000001
Partition table holds up to 128 entries
Main partition table begins at sector 2 and ends at sector 33
First usable sector is 34, last usable sector is 7518174
Partitions will be aligned on 2048-sector boundaries
Total free space is 2014 sectors (1007.0 KiB)

Number  Start (sector)    End (sector)  Size       Code  Name
   1            2048            4095   1024.0 KiB  EF02  BIOS-BOOT
   2            4096          264191   127.0 MiB   EF00  EFI-SYSTEM
   3          264192         1050623   384.0 MiB   8300  boot
   4         1050624         7518174   3.1 GiB     8300  root

```

```
# find . -name '*growfs*' 2>/dev/null 
./ostree/deploy/rhcos/deploy/808fe52200e7ac9b5fea7979234f8a849b01ff56596e0dba6e5c9ac5d10ca75d.0/usr/lib/dracut/modules.d/40ignition-ostree/ignition-ostree-growfs.service
./ostree/deploy/rhcos/deploy/808fe52200e7ac9b5fea7979234f8a849b01ff56596e0dba6e5c9ac5d10ca75d.0/usr/lib/dracut/modules.d/40ignition-ostree/ignition-ostree-growfs.sh
./ostree/deploy/rhcos/deploy/808fe52200e7ac9b5fea7979234f8a849b01ff56596e0dba6e5c9ac5d10ca75d.0/usr/lib/systemd/systemd-growfs
./ostree/deploy/rhcos/deploy/808fe52200e7ac9b5fea7979234f8a849b01ff56596e0dba6e5c9ac5d10ca75d.0/usr/sbin/xfs_growfs

```