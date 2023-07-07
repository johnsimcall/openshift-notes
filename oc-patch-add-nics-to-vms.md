I recently needed to use `kubectl patch ...` / `oc patch ...` to add an additional network interface to a Virtual Machine. Most of the patch examples I found on the internet resulted in the previous network interfaces being removed. This is the command and the accompanying `patch-file` that worked for me. The `/-` suffix on the `path:` component is the trick!

I took a lot of value from this blog post, [Kubectl Patch: What You Can Use It for and How to Do It](https://loft.sh/blog/kubectl-patch-what-you-can-use-it-for-and-how-to-do-it/). Thanks Eric Goebelbecker!

N.B. make sure the `NetworkAttachmentDefinition` exists before doing this (`oc get net-attach-def`).

```
$ oc patch vm/name-of-my-vm --type=json --patch-file add-vm-nic.patch
virtualmachine.kubevirt.io/name-of-my-vm patched
```

```
$ cat add-vm-nic.patch
- op: add
  path: /spec/template/spec/networks/-
  value:
    name: my-network-name
    multus:
      networkName: my-network-name

- op: add
  path: /spec/template/spec/domain/devices/interfaces/-
  value:
    name: my-network-name
    model: virtio
    bridge: {}
```

This is what the patch looks like in YAML
```
spec:
  template:
    spec:
      networks:
        - name: my-network-name
          multus:
            networkName: my-network-name
      domain:
        devices:
          interfaces:
            - name: my-network-name
              model: virtio
              bridge: {}
```
