# Running `virtctl usbredir ...` from an OCP node

I have a USB thingy plugged into my OCP node. I want it to be redirected to a VM on that node.

## Get `virtctl`

Easy enough, just grab a copy of it from the cluster's "Command line tools" link and extract it somewhere (`/tmp`)
```
curl https://hyperconverged-cluster-cli-download-openshift-cnv.apps.tacos.dota-lab.iad.redhat.com/amd64/linux/virtctl.tar.gz | \
  tar -C /tmp -x -z -f - virtctl
```
:::warning
:warning: CoreOS doesn't provide `lsusb`. I download it and install as a convenience in the next section
:::

## Get `usbredirect` dependency
:::danger
`virtctl usbredir` depends on `usbredirect` which comes from `usbredir-server`. I can't find that RPM being built by Red Hat for RHEL8 or RHEL9 :cry:
:::

### Build `usbredir-server` from its source RPM

:::danger
**Build it from Fedora's Source RPM** (in a container to avoid polluting my workstation)
```
$ podman run --name build-usbredir -it registry.access.redhat.com/ubi8

# rpm -ihv https://dl.fedoraproject.org/pub/fedora/linux/releases/37/Everything/source/tree/Packages/u/usbredir-0.12.0-3.fc37.src.rpm
# dnf --enablerepo=codeready-builder-for-rhel-8-x86_64-rpms install -y rpm-build meson git-core libusb1-devel glib2-devel gcc gcc-c++ gnupg2
# sed -i.backup 's/g++/gcc-c++/g' ~/rpmbuild/SPECS/usbredir.spec
# rpmbuild -ba ~/rpmbuild/SPECS/usbredir.spec
# find /root/rpmbuild/RPMS
# dnf download usbutils
# exit

$ podman cp build-usbredir:/usbutils-010-3.el8.x86_64.rpm $HOME
$ podman cp build-usbredir:/root/rpmbuild/RPMS/x86_64/usbredir-0.12.0-3.el8.x86_64.rpm $HOME
$ podman cp build-usbredir:/root/rpmbuild/RPMS/x86_64/usbredir-server-0.12.0-3.el8.x86_64.rpm $HOME
```
:::

### Install the RPMs on the RHCOS/OCP nodes

:::danger
https://access.redhat.com/solutions/4500211 says adding custom RPMs is not supported
```
$ scp $HOME/*.rpm core@node1.example.com:/tmp
$ ssh core@node1.example.com
[core@node1 ~]$ sudo rpm-ostree install /tmp/*.rpm
...
Added:
  usbredir
  usbredir-server
  usbutils

[core@node1 ~]# sudo systemctl reboot
```

:::

## Setup KUBECONFIG on the OCP node

:::warning
:warning: I don't like the idea of using system:admin / cluster-admin permissions for this
:::

```
$ ssh core@node1.example.com
[core@node1 ~]$ sudo -i
[root@node1 ~]# export KUBECONFIG=/etc/kubernetes/static-pod-resources/kube-apiserver-certs/secrets/node-kubeconfigs/localhost.kubeconfig
[root@node1 ~]# oc whoami
system:admin
```

## Make sure the VM is configured to accept redirected USB devices

:::warning
:warning: Your VM needs to have an empty [`clientPassthrough: {}`](http://kubevirt.io/api-reference/v0.44.0/definitions.html#_v1_clientpassthroughdevices) otherwise you'll get an error like this:

"Can't access VMI rhel8-happy-golucky: Operation cannot be fulfilled on virtualmachineinstance.kubevirt.io "rhel8-happy-golucky": Not configured with USB Redirection"
:::

```
oc patch -n jcall vm/rhel8-happy-golucky --type=merge -p \
  '{"spec":{"template":{"spec":{"domain":{"devices":{"clientPassthrough": {}}}}}}}'
```

## Run `virtctl`

:::info
:information_source: You need to know the USB Vendor and Product ID for this next part. `lsusb` can you help you find the right numbers (`067b:2303` is my serial adapter thingy)
:::

```
[root@node1 ~]# /tmp/virtctl -n jcall usbredir 067b:2303 rhel8-happy-golucky &
{"component":"","level":"info","msg":"port_arg: 'localhost:41949'","pos":"usbredir.go:157","timestamp":"2022-12-03T10:22:39.583643Z"}
{"component":"","level":"info","msg":"args: '[--device 067b:2303 --to localhost:41949]'","pos":"usbredir.go:158","timestamp":"2022-12-03T10:22:39.583747Z"}
{"component":"","level":"info","msg":"Executing commandline: 'usbredirect [--device 067b:2303 --to localhost:41949]'","pos":"usbredir.go:159","timestamp":"2022-12-03T10:22:39.584022Z"}
{"component":"","level":"info","msg":"Connected to usbredirect at 303.988732ms","pos":"usbredir.go:126","timestamp":"2022-12-03T10:22:39.887600Z"}
```

# Acknowledgements
I stumbled upon this [brief note from Victor Toso](https://groups.google.com/g/kubevirt-dev/c/ZuAYp6twYp8/m/P-CfzR-0AgAJ) that helped me get going. Martin Tessun also said something that got me question my previous method. My previous method was to buy a special [PCIe USB card](https://www.startech.com/en-us/cards-adapters/pexusb3s24), write custom scripts to unhook the card from `xhci_hcd` and give it to `vfio-pci`, and then configure the VM to use PCI passthrough. The nice thing about PCI passthrough was that the VM wouldn't start until it found a card it could use. If I continue with this USB redirect method I will have to figure out where the VM is running/scheduled and redirect the device after the VM has started. Oh, and then there's the matter of making CoreOS unsupported because of the extra RPMs I had to build & install...

# Can we automate everything?

:shrug: IDK
