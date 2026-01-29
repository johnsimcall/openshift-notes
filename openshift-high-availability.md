---
title: High Availability for OpenShift workloads
tags: [openshift]

---

# High Availability for OpenShift workloads

[![hackmd-github-sync-badge](https://hackmd.io/suvYTjxASQ-1o01fho1hYw/badge)](https://hackmd.io/suvYTjxASQ-1o01fho1hYw)


I like to run Virtual Machines (VMs) and container-based workloads on my bare metal OpenShift cluster. Sometimes the bare metal `Nodes` in my OpenShift cluster crash. Sadly, all of the `VMs` & `Pods` that were running on the Node that crashed stop working too. OpenShift will restart some of the Pods on other Nodes, but the VMs and certain other Pods never get restarted. Instead, the output of `oc get pods` shows my VMs and Pods are stuck `Terminating` forever.

This notes describes why some Pods & VMs don't get restarted, and how to add the missing pieces to make OpenShift restart everything.

## Explain why some Pods & VMs don't restart automatically

I'd like to explain why the VMs and certain Pods don't get restarted automatically. It's because of the Storage (`PersistentVolumes` / `PersistentVolumeClaims`) that the Pods and VMs use. If a Pod is configured to use `ReadWriteOnce` storage, it can't be automatically restarted on another Node for safety reasons. And VMs can't be automatically restarted either, even if they use `ReadWriteMany` storage. For safety reasons, OpenShift & OpenShift Virtualization will refuse to restart these workloads until something confirms that the Node is powered off -- which means the VMs and Pods can be safely restarted.

Normal filesystems like `XFS`, `NTFS`, & `ext4` expect to be used by one (and only one) system at a time. Some people call this "at-most-one". Can you imagine two laptops connected to one hard drive (and one filesystem) at the same time? It's ridiculous because the "shared" filesystem would immediately become corrupted because each laptop would be updating the filesystem's internal structures and overwritting the entries of the other laptop.

:::info
A few very smart people have created "clustered filesystems", but those don't apply in this situation. If you're curious, consider researching [StorNext FS](https://en.wikipedia.org/wiki/StorNext_File_System), [IBM's GPFS](https://en.wikipedia.org/wiki/GPFS), [and others](https://en.wikipedia.org/wiki/Clustered_file_system).

Please note, a "network" filesystem like NFS and SMB/CIFS is not the same as a "clustered" filesystem
:::

## How to enable High Availability

The term "High Availability" or HA in this case means restarting workloads as soon as possible in order to reduce the amount of time your applications are `unavailable.` When a Node in your OpenShift cluster crashes its `kubelet` agent stops sending hearbeat messages (sent every 10 seconds by default). If enough of these updates are missed (4 by default), OpenShift will flag the `Node` as `NotReady` and stop assigning new Pods & VMs to it. OpenShift can't know for sure what caused the Node to stop sending heartbeat messages. It's possible that the Node is still running, but it was disconnected from the heartbeat network. It's possible that the `kubelet` agent crashed, but other jobs are still running on the Node. Or the Node could have lost power, or the Operating System could have panicked ([a Linux Blue Screen of Death](https://www.phoronix.com/news/Linux-DRM-Panic-BSoD-Picture)) which halts all processes. Because OpenShift can't know for sure if the VMs & Pods are stopped, and knowing that data will be corrupted if the VM is running in more than one place at a time (two laptops, one filesystem), then OpenShift does the only safe thing it can do -- nothing.

## Telling OpenShift its safe to restart workloads

The problem of ensuring that a workload (Pod or VM) only runs "at most once" has already been solved using "fence" agents. Fencing a node refers to the process of moving the node out of the cluser, usually by powering off the server, or disconnecting the network. Running a fence agents when a Node stops sending heartbeat messages is known as "Remediating" the node. Remediating the node in this case is really just [turning it off and back on again](https://youtu.be/rksCTVFtjM4?si=Yrlo7ryBzkQVVMrG&t=37). Reboots fix 92.8% of all computer problems, right? :)




```yaml=
---
kind: Secret
apiVersion: v1
metadata:
  name: all-nodes
  namespace: openshift-workload-availability
stringData:
  --action: reboot
  --ssl-insecure: ''


---
kind: Secret
apiVersion: v1
metadata:
  name: node-01   # also create this secret for node-02, node-03, etc...
  namespace: openshift-workload-availability
stringData:
  --ip: node-01-idrac.mgmt-vlan.example.com  # change for node-02, etc...
  --username: administrator
  --password: MySecretPassword123
  


---
apiVersion: fence-agents-remediation.medik8s.io/v1alpha1
kind: FenceAgentsRemediationTemplate
metadata:
  name: reboot-dell-nodes-via-idrac-redfish
  namespace: openshift-workload-availability
spec:
  template:
    spec:
      agent: fence_redfish
      nodeSecretNames:
        #name_of_node: name_of_secret
        node-01: node-01
        node-02: node-02
        node-03: node-03
      sharedSecretName: all-nodes  #common fence-agent parameters 
      remediationStrategy: OutOfServiceTaint
      retrycount: 5
      retryinterval: 5s
      timeout: 2m0s     # wait this long for the reboot command to be accepted


---
apiVersion: remediation.medik8s.io/v1alpha1
kind: NodeHealthCheck
metadata:
  name: worker-nodes-health-check
spec:
  selector:
    matchExpressions:   # only check and repair worker nodes
      - key: node-role.kubernetes.io/worker
        operator: Exists
        values: []
  remediationTemplate:
    apiVersion: fence-agents-remediation.medik8s.io/v1alpha1
    kind: FenceAgentsRemediationTemplate
    name: reboot-dell-nodes-via-idrac-redfish
    namespace: openshift-workload-availability
  minHealthy: 51%       # don't reboot anything, unless 51% or more of the nodes are online
  unhealthyConditions:
    - duration: 300s    # wait 5 minutes before rebooting the node
      status: 'False'
      type: Ready
    - duration: 300s    # wait 5 minutes before rebooting the node
      status: Unknown
      type: Ready

```


## Links

https://github.com/ClusterLabs/fence-agents/tree/main/agents
https://docs.redhat.com/en/documentation/workload_availability_for_red_hat_openshift/
# High Availability for OpenShift workloads

[![hackmd-github-sync-badge](https://hackmd.io/suvYTjxASQ-1o01fho1hYw/badge)](https://hackmd.io/suvYTjxASQ-1o01fho1hYw)


I like to run Virtual Machines (VMs) and container-based workloads on my bare metal OpenShift cluster. Sometimes the servers in my OpenShift cluster crash, and the VMs & Pods crash too. Some of my Pods get restarted, but the VMs and some other Pods never get restarted. Instead, the output of `oc get pods` shows my VMs and Pods are stuck `Terminating` forever.

This notes describes why some Pods & VMs don't get restarted, and how to add the missing pieces to make OpenShift restart everything.

## Explain why VMs don't 

```yaml=
---
kind: Secret
apiVersion: v1
metadata:
  name: all-nodes
  namespace: openshift-workload-availability
stringData:
  --action: reboot
  --ssl-insecure: ''


---
kind: Secret
apiVersion: v1
metadata:
  name: node-01   # also create this secret for node-02, node-03, etc...
  namespace: openshift-workload-availability
stringData:
  --ip: node-01-idrac.mgmt-vlan.example.com  # change for node-02, etc...
  --username: administrator
  --password: MySecretPassword123
  


---
apiVersion: fence-agents-remediation.medik8s.io/v1alpha1
kind: FenceAgentsRemediationTemplate
metadata:
  name: reboot-dell-nodes-via-idrac-redfish
  namespace: openshift-workload-availability
spec:
  template:
    spec:
      agent: fence_redfish
      nodeSecretNames:
        #name_of_node: name_of_secret
        node-01: node-01
        node-02: node-02
        node-03: node-03
      sharedSecretName: all-nodes  #common fence-agent parameters 
      remediationStrategy: OutOfServiceTaint
      retrycount: 5
      retryinterval: 5s
      timeout: 2m0s     # wait this long for the reboot command to be accepted


---
apiVersion: remediation.medik8s.io/v1alpha1
kind: NodeHealthCheck
metadata:
  name: worker-nodes-health-check
spec:
  selector:
    matchExpressions:   # only check and repair worker nodes
      - key: node-role.kubernetes.io/worker
        operator: Exists
        values: []
  remediationTemplate:
    apiVersion: fence-agents-remediation.medik8s.io/v1alpha1
    kind: FenceAgentsRemediationTemplate
    name: reboot-dell-nodes-via-idrac-redfish
    namespace: openshift-workload-availability
  minHealthy: 51%       # don't reboot anything, unless 51% or more of the nodes are online
  unhealthyConditions:
    - duration: 300s    # wait 5 minutes before rebooting the node
      status: 'False'
      type: Ready
    - duration: 300s    # wait 5 minutes before rebooting the node
      status: Unknown
      type: Ready

```


## Links

https://github.com/ClusterLabs/fence-agents/tree/main/agents
