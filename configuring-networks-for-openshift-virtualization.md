# OpenShift Network Configuration for VMs

A brief note here about setting up the physical and virtual networking so that OpenShift can connect VMs directly to their appropriate network segment / VLAN. This is documented in the [official docs](https://docs.openshift.com/container-platform/4.16/virt/vm_networking/virt-networking-overview.html) as well...

If you don't setup VM networks, the VMs are required to use the default "Pod network" which makes SSH and RDP connections awkward.

![Illustration of VMs using a direct network connection](https://hackmd.io/_uploads/rk8GxIQCR.png)

(7) Secondary VM networks are typically bridged directly to a physical network, with or without VLAN encapsulation.
(8) Secondary VM networks typically use a dedicated set of NICs
(*) Other configurations are possible, but out-of-scope for this note

## Configure the Nodes / Hypervisors

I prefer to use the declarative syntax of NMState to create the hosts' `bonds` and `bridges`. This is done by creating `NodeNetworkConfigurationPolicies`.
You must install the `Kubernetes NMState Operator` in order to apply the YAML below.
You can also create this via the WebUI. I'll provide an image below...

```yaml=
---
apiVersion: nmstate.io/v1
kind: NodeNetworkConfigurationPolicy
metadata:
  name: vm-bridge
spec:
# Only use nodeSelector if:
#   * You're using Static IPs for some reason (storage)
#   * Your NIC names are inconsistent between servers
#  nodeSelector:
#    kubernetes.io/hostname: node1.example.com
#    node-role.kubernetes.io/worker: ""
  desiredState:
    interfaces:
      - name: vm-bridge
        description: Bridge for VMs to directly connect with networksVLANs via NetworkAttachmentDefinitions
        state: up
        type: linux-bridge
        #mtu: 9000            ### optional, your switch must be configured for jumbo frames
        bridge:
          options:
            stp:
              enabled: true
          port:
            - name: ens1f1    ### this needs to be changed
            #- name: vm-bond  ### optional, created below
        ipv4:
          enabled: false      ### the host doesn't normally need an IP address
        ipv6:
          enabled: false


      - name: vm-bond
        description: LACP or active-backup bond for VLANs 676-South-VMs & 677-West-VMs
        state: up
        type: bond
        #mtu: 9000             ### your switch must be configured for jumbo frames
        link-aggregation:
          mode: active-backup
          #mode: 802.3ad       ### the switch must be configured for LACP/link-agg/port-channel
          port:
            - ens1f1
        ipv4:
          enabled: false       ### the host doesn't normally need an IP address
        ipv6:
          enabled: false
        lldp:
          enabled: true        ### optional, show lldp using "nmcli dev lldp"
```

Don't delete the NNCP if you make a mistake and want to remove the previously applied configurations. Instead, modify the YAML and set the interface `state:` to `absent` like this:

```yaml=
## CLEANUP
---
apiVersion: nmstate.io/v1
kind: NodeNetworkConfigurationPolicy
metadata:
  name: cleanup
spec:
  nodeSelector:
    #kubernetes.io/hostname: node1.example.com
    #node-role.kubernetes.io/worker: ''
  desiredState:
    interfaces:
    - name: vm-bridge
      type: linux-bridge
      state: absent
    - name: vm-bond
      type: bond
      state: absent
```

![Screenshot - Creating the host/node config via the Web UI](https://hackmd.io/_uploads/SyYr-8QR0.png)

The Web UI will show a green check mark when the configuration has been applied to the node(s)

![Screenshot - Succesfully configured all nodes](https://hackmd.io/_uploads/ByHxHIQAR.png)


## Create the virtual networks

Now that the server/host/node(s) have their physical interfaces and bonds added to a bridge, it's time to create the "vSwitch" / "Distributed Port Group". In other words, what OpenShift calls a `NetworkAttachmentDefinition`. Please be aware that `net-attach-defs` (the short name, which is better than `NADs`) must be created in a particular project. Please avoid creating these in the `default` namespace/Project.

The WebUI can be used to create these too. I'll provide an image below.

### Allow the VMs to use the bridge(s) with the default/native VLAN

```yaml=
---
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
  name: native-vlan
  namespace: vm-test       ### must exist in the same namespace as the VM
  annotations:
    description: Native VLAN of vm-bond/vm-bridge (eth0 & eth1)
    k8s.v1.cni.cncf.io/resourceName: bridge.network.kubevirt.io/vm-bridge
spec:
  config: '{
    "name": "native-vlan",
    "bridge": "vm-bridge",
    "type": "cnv-bridge",
    "cniVersion": "0.3.1",
    "macspoofchk": true,
    "ipam": {},
    "preserveDefaultVlan": false
  }'
```

![Screenshot - VM traffic on the native/default VLAN](https://hackmd.io/_uploads/r17IMLm00.png)

### Allow the VMs to use the bridge(s) with a specific VLAN

TODO: Add a blurb here about OpenShift being responsible for adding / removing VLAN tags to the VMs traffic.

```yaml=
---
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
  name: vlan-1234
  namespace: vm-test       ### must exist in the same namespace as the VM
  annotations:
    description: VLAN 1234 trunked to vm-bond/vm-bridge (eth0 & eth1)
    k8s.v1.cni.cncf.io/resourceName: bridge.network.kubevirt.io/vm-bridge
spec:
  config: '{
    "name": "vlan-1234",
    "vlan": 1234,
    "bridge": "vm-bridge",
    "type": "cnv-bridge",
    "cniVersion": "0.3.1",
    "macspoofchk": true,
    "ipam": {},
    "preserveDefaultVlan": false
  }'
```

![Screenshot - VM traffic is tagged/assigned to VLAN 1234](https://hackmd.io/_uploads/HJOjM87RC.png)


### Allow VMs to share the "primary" OpenShift NIC

NOTE 1: It's a better idea to use "secondary" NICs instead of forcing VMs to share OpenShift's `br-ex` primary interface.
NOTE 2: Creating this via the GUI creates an empty annotation that prevents VMs from scheduling. You must remove the annotation in order for the VM to boot.

```yaml=
# You must make three changes to this file
# 1 - choose a name
# 2 - choose a namespace
# 3 - update the "netAttachDefName" value to be "<namespace>/<name>"

# Please note:
# "physnet" is automatically created by OVNKubernetes during the installation of OpenShift

---
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
  name: share
  namespace: vm-test
spec:
  config: |2
    {
            "name": "physnet",
            "topology":"localnet",
            "netAttachDefName": "vm-test/share",
            "type": "ovn-k8s-cni-overlay",
            "cniVersion": "0.4.0"
    }
```

![Screenshot - VMs share OpenShift's primary NIC](https://hackmd.io/_uploads/S1dZXUmA0.png)


Thanks for reading. Nothing else to see here...

:::spoiler

# Rubbish beyond here

## Oddly specific example

The interface (enp5s0f0) is able to use the default/native vlan (921) configured on the switch for OpenShift.
An additional Storage VLAN (923) is available on the same interface (enp5s0f0)

The `NodeNetworkConfigurationPolicies` to create a vlan-interface and assign a static IP address would like like this.
**Please note**: Because static IP addresses are being used, these `NNCPs` must use a `nodeSelector`. Every node needs its own `NNCP` with a unique static IP address.

```yaml=
---
apiVersion: nmstate.io/v1
kind: NodeNetworkConfigurationPolicy
metadata:
  name: storage-node-1                          #change
spec:
  nodeSelector:
    kubernetes.io/hostname: node-1.example.com  #change
  desiredState:
    interfaces:
    - description: VLAN 923 (Storage)
      name: enp5s0f0.923                        #match with lines 25-26
      type: vlan
      state: up      
      #mtu: 9000   #default is 1500             #confirm
      ipv4:
        enabled: true
        dhcp: false
        address:
        - ip: 192.1.196.21                      #confirm + change
          prefix-length: 24                     #confirm
      ipv6:
        enabled: false
      vlan:
        base-iface: enp5s0f0                    #confirm
        id: 923                                 #confirm

---
apiVersion: nmstate.io/v1
kind: NodeNetworkConfigurationPolicy
metadata:
  name: storage-node-2                          #change
spec:
  nodeSelector:
    kubernetes.io/hostname: node-2.example.com  #change
  <...snip...>
  
---
---
apiVersion: nmstate.io/v1
kind: NodeNetworkConfigurationPolicy
metadata:
  name: storage-node-3                          #change
spec:
  nodeSelector:
    kubernetes.io/hostname: node-3.example.com  #change
  <...snip...>
```

:::