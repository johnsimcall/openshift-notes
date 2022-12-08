# Running `virtctl usbredir ...` from an OCP node

I have some USB things plugged into my OCP node. I want them to be redirected to a VM on that node. I describe two methods below to run the redirection software, 1) install the RPM [directly into RHCOS](#Install-the-RPMs-on-the-RHCOS/OCP-nodes) (:warning: this is not supported.) and 2) install the RPM [into a Pod/container image](#can-we-run-everything-in-a-pod?).

## Get `virtctl`

Easy enough, just grab a copy of it from the cluster's "Command line tools" link and extract it somewhere (`/tmp`)
```
curl https://hyperconverged-cluster-cli-download-openshift-cnv.apps.tacos.dota-lab.iad.redhat.com/amd64/linux/virtctl.tar.gz | \
  tar -C /tmp -x -z -f - virtctl
```
:::warning
:warning: CoreOS doesn't provide `lsusb`. I install it and `/etc/udev/hwdb.bin` from the `systemd-udev` RPM in the next section.
:::

## Get `usbredirect` dependency
:::danger
`virtctl usbredir` depends on `usbredirect` which comes from `usbredir-server`. I can't find that RPM from any RHEL8 or RHEL9 repos :cry:
:::

*(This seems like a oversight, we should reach out to the CNV folks)*

### Build `usbredir-server` from its source RPM

:::danger
**Build it from RHEL8's source RPM** (in a container to avoid polluting my workstation)
```
#Containerfile (run on a RHEL8 host to authorize non-UBI repos)
# podman build -t quay.io/jcall/ubi8-usbredir -f ./Containerfile

# build environment
FROM registry.access.redhat.com/ubi8 AS builder
RUN dnf download --source usbredir
RUN dnf download usbutils
RUN dnf -y builddep ./usbredir-* --enablerepo=codeready-builder-for-rhel-8-x86_64-rpms 
RUN dnf -y install rpm-build gcc gcc-c++
RUN rpm -ihv ./usbredir*.src.rpm && rm ./usbredir*.src.rpm
RUN rpmbuild -ba /root/rpmbuild/SPECS/usbredir.spec 
RUN ln -s /root/rpmbuild/RPMS/x86_64/usbredir-[0-9]* / && \
    ln -s /root/rpmbuild/RPMS/x86_64/usbredir-server-[0-9]* /

# execution environment
FROM registry.access.redhat.com/ubi8
WORKDIR /root
COPY --from=builder /*.rpm .
RUN dnf -y install *.rpm && dnf clean all && rm -v *.rpm
COPY ./oc /usr/local/bin
COPY ./virtctl /usr/local/bin
RUN  mkdir -p /etc/udev
COPY ./hwdb.bin /etc/udev/
COPY ./Containerfile-rpmbuild-usbredir /root/Containerfile
```

:arrow_down_small: rubbish below :arrow_down_small:
```
$ podman run --name build-usbredir --hostname usbredir -it registry.access.redhat.com/ubi8

# rpm -ihv https://dl.fedoraproject.org/pub/fedora/linux/updates/37/Everything/source/tree/Packages/u/usbredir-0.13.0-1.fc37.src.rpm
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

### Alternative ([see below](#can-we-run-everything-in-a-pod?)): Run a privileged container with usbredirect in it

(We might not even need full privileges for the container, I am not sure what is required to access usb devices)

:construction: We *should* be able to run either podman as a system service or better yet as a Pod. 

#### DaemonSet/Deployment/StatefulSet

If we used a DaemonSet we may need some sort of locking mechanism to ensure that we are only redirecting usb exactly once among all the pods. This can be done simply with ConfigMaps which is how multiple controllers in Kubernetes maintain single-node leadership.

If we used a Deployment or a StatefulSet we would have slightly better control over how many pods are running at once.

#### Container Command/Args

Whatever way we choose to run the Pod we will need to have some logic in the usbredirect container to:

- handle any sort of locking, if required
- start and manage the virtctl invocation

What is nice though is that this script can just be baked into the container image. We can pass through the USB vendor/product ID through env vars.

#### Pod Authentication

If we use a Pod we can use the service account credentials and can get rid of the next section :arrow_down: 

## Setup KUBECONFIG on the OCP node

:::warning
:warning: I don't like the idea of using system:admin / cluster-admin permissions for this
:::

:::danger
TODO: create a `Role` to restrict permissions.
:::

```
$ ssh core@node1.example.com
[core@node1 ~]$ sudo -i
[root@node1 ~]# export KUBECONFIG=/etc/kubernetes/static-pod-resources/kube-apiserver-certs/secrets/node-kubeconfigs/localhost.kubeconfig
[root@node1 ~]# oc whoami
system:admin
```

Better than using system:admin, we can add a Role with the permissions we need and then a RoleBinding for the nodes in the cluster (system:nodes)

```
sh-4.4# export KUBECONFIG=/var/lib/kubelet/kubeconfig 
sh-4.4# oc whoami
system:node:hp1
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

# Run `virtctl usbredir ...` in a container

Make sure the previously built `usbredir` RPMs are available, and `lsusb`. Then add the `oc` client and the `virtctl` client from your cluster's client download links.

## Build our container image

```
---
kind: BuildConfig
apiVersion: build.openshift.io/v1
metadata:
  name: usb-build-image
  namespace: jcall
spec:
  mountTrustedCA: true
  strategy:
    type: Docker
  source:
    dockerfile: |
      FROM registry.access.redhat.com/ubi8

      ENV OC_URL=https://downloads-openshift-console.apps.tacos.dota-lab.iad.redhat.com/amd64/linux/oc.tar                                            
      ENV VIRTCTL_URL=https://hyperconverged-cluster-cli-download-openshift-cnv.apps.tacos.dota-lab.iad.redhat.com/amd64/linux/virtctl.tar.gz         

      RUN curl -s $OC_URL | tar -C /usr/local/bin -x oc
      RUN curl -s $VIRTCTL_URL | tar -C /usr/local/bin -xz virtctl

      RUN dnf -y install http://rhdata6/usbredir-0.12.0-3.el8.x86_64.rpm \
                         http://rhdata6/usbredir-server-0.12.0-3.el8.x86_64.rpm && \                                                                  
          dnf clean all
  output:
    to:
      kind: ImageStreamTag
      name: 'virtctl-usbredir:latest'
  triggers:
    - type: "ConfigChange"

---
kind: ImageStream
apiVersion: image.openshift.io/v1
metadata:
  name: virtctl-usbredir
  namespace: jcall
```

## Allow the ServiceAccount to use the `privileged` SecurityContextConstraint. Also allow the Pod to checkout a wildly powerful Token!

```
---
kind: ServiceAccount
apiVersion: v1
metadata:
  name: virtctl-serviceaccount
  namespace: jcall

---
#allow `oc` in the pod to get nodes, etc...
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: virtctl-serviceaccount-token
roleRef:
  kind: ClusterRole
  apiGroup: rbac.authorization.k8s.io
  name: cluster-admin
subjects:
  - kind: ServiceAccount
    apiGroup: ""
    name: virtctl-serviceaccount
    namespace: jcall

---
#allow `virtctl` to run as root, use the host network, and mount /dev in the container
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: virtctl-serviceaccount-privileged
roleRef:
  kind: ClusterRole
  apiGroup: rbac.authorization.k8s.io
  name: system:openshift:scc:privileged
subjects:
  - kind: ServiceAccount
    name: virtctl-serviceaccount
    namespace: jcall
```

## Deploy the Pod

```
---
kind: Deployment
apiVersion: apps/v1
metadata:
  namespace: jcall
  name: virtctl-test
  labels:
    app: virtctl-test
spec:
  replicas: 1
  selector:
    matchLabels:
      app: virtctl-test
  template:
    metadata:
      labels:
        app: virtctl-test
    spec:
      serviceAccountName: virtctl-serviceaccount
      nodeName: rhdata1.dota-lab.iad.redhat.com #todo: use node labels to figure out where i plugged in my usb
      #vm.kubevirt.io/name=rhel8-sore-aardvark  #set affinity to so this pod runs on the same host as the VM, or at least on the host with the USB
      terminationGracePeriodSeconds: 0  # remove this later
      containers:
        - name: virtctl
          image: image-registry.openshift-image-registry.svc:5000/jcall/virtctl-usbredir:latest
          securityContext:
            privileged: true
          volumeMounts:
            - name: dev
              mountPath: /dev
          command: [ "sleep", "1h" ]
      volumes:
      - name: dev
        hostPath:
          path: /dev
```

## TODO: Run a command in the Pod

I have a cheap USB3 memory stick (13fe:6300) plugged into node-1

```
oc exec -it deployment/virtctl-test -- /bin/bash
virtctl usbredir 13fe:6300 rhel8-sore-aardvark

# make two commands into one!
oc exec deployment/virtctl-test -- virtctl usbredir 13fe:6300 rhel8-sore-aardvark
```
