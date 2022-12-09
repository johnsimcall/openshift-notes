# Connecting USB devices to Virtual Machines with OpenShift Virtualization

I have some USB devices plugged into my baremetal OpenShift servers. I want those USB devices to be connected/passed-through/redirected to a Virtual Machine running on one of the server/node/hypervisors. At first I purchased a very specific [PCI Express card that adds USB ports](https://www.startech.com/en-us/cards-adapters/pexusb3s24) to the server and used [PCI passthrough](https://docs.openshift.com/container-platform/4.11/virt/virtual_machines/advanced_vm_management/virt-configuring-pci-passthrough.html) to connect it (and the USB devices plugged into it) to the VM. It worked, but it required some awkward steps to unbind the card from the kernel's built-in `xhci_hcd` USB driver/module and rebind to the `vfio-pci` module. Another potential solution could use a [KubeVirt "hook sidecar"](https://github.com/kubevirt/kubevirt/tree/main/cmd/example-hook-sidecar) to add libvirt XML like "<hostdev mode='subsystem' type='usb'> ..." But that seems too difficult for me to accomplish at the moment. That leaves me with the "redirect" method provided by `virtctl`, otherwise known as `virtctl usbredir ...` **Here are my notes about using `virtcl usbredir` in a `Pod`...**

## Create a container image with `virtctl` and `lsusb`

I'm running my Virtual Machines with OpenShift Virtualization which means my servers/nodes/hypervisors are installed with [Red Hat Enterprise Linux Core OS (RHEL CoreOS or RHCOS for short.)](https://docs.openshift.com/container-platform/4.11/architecture/architecture-rhcos.html) This means that we avoid installing missing RPMs (like `usbredir-server`) in the OS. Instead we create container images that have the apps/utilities/RPMs we need and run those containers as Pods. So step one is to build a container image that has everything we need.
    
:::danger
`virtctl usbredir` depends on `/usr/bin/usbredirect` which is provided by the `usbredir-server` RPM. I've been told that RPM will become availabe with RHEL9.2. Until then I'll compile my own RPM using the RHEL8 sources

    TODO: why not just build it from the RHEL9 sources?
:::

### Build `usbredir-server` from its source RPM

Here are the steps I followed to build the `usbredir-server` RPM in a container, and then push a sanitized container image to quay.io The `Containerfile` (aka `Dockerfile`) below has two sections. The fist section `FROM .../ubi8 AS builder` downloads a bunch of compilers, utilities, and dependencies that we don't need when we're running the program. So the second `FROM .../ubi8` section copies out the RPMs we needs, and discards the stuff from the first section. I appreciate that everything can go in just one `Containerfile`. Yeah simplicity!

:::warning
Build/run this `Containerfile` on a subscribed/entitled RHEL8 machine to entitle the necessary repos inside the container. The UBI8 repos don't have everything we need.
:::
    
:::warning
In addition to the `Containerfile` we also need to put a copy of the `virtctl` binary in the same directory. I also add in the `hwdb.bin` file because that's what allows `lsusb` to add friendly names to the USB devices it finds. I also add in the `oc` binary just for troubleshooting and debugging (e.g. `oc get vm`)
```
# oc get ConsoleCLIDownload oc-cli-downloads virtctl-clidownloads-kubevirt-hyperconverged -o yaml | grep -e '/amd64/linux/'
    - href: https://downloads-openshift-console.apps.my-cluster.example.com/amd64/linux/oc.tar
    - href: https://hyperconverged-cluster-cli-download-openshift-cnv.apps.my-cluster.example.com/amd64/linux/virtctl.tar.gz
    
curl oc...      | tar -xv oc
curl virtctl... | tar -xzv virtctl

cp -v /etc/udev/hwdb.bin .
    
# ls .
Containerfile
oc
virtctl
```
:::

#### Containerfile

Create this `Containerfile` in a directory with the `oc`, `virtctl`, and `hwdb.bin` files
```
# temporary build environment, gets discarded
FROM registry.access.redhat.com/ubi8 AS builder
RUN dnf download --source usbredir
RUN dnf download usbutils
RUN dnf -y builddep ./usbredir-* --enablerepo=codeready-builder-for-rhel-8-x86_64-rpms 
RUN dnf -y install rpm-build gcc gcc-c++
RUN rpm -ihv ./usbredir*.src.rpm && \
    rm ./usbredir*.src.rpm
RUN rpmbuild -ba /root/rpmbuild/SPECS/usbredir.spec 
RUN ln -s /root/rpmbuild/RPMS/x86_64/usbredir-[0-9]* / && \
    ln -s /root/rpmbuild/RPMS/x86_64/usbredir-server-[0-9]* /

# minimal execution environment, gets pushed to quay.io
FROM registry.access.redhat.com/ubi8
WORKDIR /root
COPY --from=builder /*.rpm .
RUN dnf -y install *.rpm && \
    dnf clean all && \
    rm -v *.rpm
COPY ./oc /usr/local/bin
COPY ./virtctl /usr/local/bin
RUN  mkdir -p /etc/udev
COPY ./hwdb.bin /etc/udev/
COPY ./Containerfile-rpmbuild-usbredir /root/Containerfile
```

Now build `usbredir`, `usbredir-server` and the container image with this command
```
podman build -t quay.io/your-name-here/ubi8-usbredir -f ./Containerfile
```

After the container has been built, push it to quay.io with this command
```
podman login quay.io
podman push quay.io/your-name-here/ubi8-usbredir
```

## Run `virtctl usbredir ...` in a privileged Pod

:::danger
I don't like the idea of using `priviliged` permissions and `cluster-admin` authorization.

    TODO: create a `SCC` to restrict pod/container permissions
    TODO: create a `Role` to restrict authentication & authorization.
:::
    
### Pod permissions

By default `Pods` created by OpenShift use a `ServiceAccount` that is assigned to the `restricted` [`SecurityContextConstraint`](https://docs.openshift.com/container-platform/4.11/authentication/managing-security-context-constraints.html). Among other things, the `restricted` `SCC` prevents `Pods` from running as `root` and mounting host filesystems like `/dev`. We need to grant more permissions to our `Pod` so that `virtcl usbredir` can function. 
    
### Pod authentication & authorization

`Pods` that run in OpenShift are allowed to use a token associated with their `ServiceAccount` to talk to the OpenShift API. The default token doesn't authorize much of anything. For example, you can't run `oc get vms` in the Pod. We need to increase the authorization of the `ServiceAccount`

#### Allow the ServiceAccount to use `privileged` permissions and have `cluster-admin` authorization!
    
Save the three YAML sections belwo to a file and then create the resources using
```
oc create -f your-filename.yaml
```

```
---
# create a new ServiceAccount
kind: ServiceAccount
apiVersion: v1
metadata:
  name: virtctl-serviceaccount
  namespace: your-namespace

---
# give the ServiceAccount permission to use the "restricted" SecurityContextConstraint (e.g. run as root and mount /dev in the container)
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: virtctl-serviceaccount-privileged-permissions
roleRef:
  kind: ClusterRole
  apiGroup: rbac.authorization.k8s.io
  name: system:openshift:scc:privileged
subjects:
  - kind: ServiceAccount
    name: virtctl-serviceaccount
    namespace: your-namespace

---
# authentication and authorization for `oc` and `virtctl` in the pod to "oc get nodes", "oc get vms", etc...
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: virtctl-serviceaccount-token-authorization
roleRef:
  kind: ClusterRole
  apiGroup: rbac.authorization.k8s.io
  name: cluster-admin
subjects:
  - kind: ServiceAccount
    apiGroup: ""
    name: virtctl-serviceaccount
    namespace: your-namespace
```

## Deploy the Pod

Here is an example `Deployment` that will launch a `privileged` `Pod` from the container image we built. This example has no logic. After the `Pod` starts up you need to run `virtctl usbredir ...` manually.

:::info
TODO: Use `nodeSelector` or `nodeAffinity` instead of `nodeName` below. Ideally the `VirtualMachine` and the `virtctl-usbredir` `Pod` both run on the host that has the USB device to avoid adding network latency to the redirection.
:::

```
---
kind: Deployment
apiVersion: apps/v1
metadata:
  namespace: your-namespace                            # change this
  name: virtctl-usbredir
  labels:
    app: virtctl-usbredir
spec:
  replicas: 1
  selector:
    matchLabels:
      app: virtctl-usbredir
  template:
    metadata:
      labels:
        app: virtctl-usbredir
    spec:
      nodeName: node1.example.com                      # change this
      serviceAccountName: virtctl-serviceaccount
      containers:
        - name: virtctl-usbredir
          image: quay.io/your-name-here/ubi8-usbredir  # change this
          securityContext:
            privileged: true
          volumeMounts:
            - name: dev
              mountPath: /dev
          command: [ "sleep", "365d" ]
      volumes:
      - name: dev
        hostPath:
          path: /dev
```

Wait a minute or two and then make sure the `Pod` is `Running` like this
```
# oc get pods -l app=virtctl-usbredir
NAME                                READY   STATUS    RESTARTS   AGE
virtctl-usbredir-7865dd4fc8-fxxd6   1/1     Running   0          2d

```

## Make sure your VM is configured to accept redirected USB devices

:warning: Your VM needs to have an empty [`clientPassthrough: {}`](http://kubevirt.io/api-reference/v0.44.0/definitions.html#_v1_clientpassthroughdevices) under `spec.template.spec.domain.devices` otherwise you'll get this error when you run `virtctl usbredir ...`

:::danger
Can't access VMI rhel8-happy-golucky: Operation cannot be fulfilled on virtualmachineinstance.kubevirt.io "rhel8-happy-golucky": Not configured with USB Redirection
:::

You can easily add the missing section to your VM, and then reboot your VM, with the following commands (assuming your VM is configured for `running: true` or `runStrategy: Always`) You have to reboot your VM for the change to be recognized.
```
oc patch vm/rhel8-happy-golucky --type=merge -p \
  '{"spec":{"template":{"spec":{"domain":{"devices":{"clientPassthrough": {}}}}}}}'

oc delete vmi/rhel8-happy-golucky
```

## Run `virtctl usbredir`

We're so close to victory! Now all you need to do is login to the `Pod`, identify your USB device's Vendor and Product ID, and run `virtctl usbredir ...`! You can log into the `Pod` by running this command
```
oc exec deployment/virtctl-usbredir -- /bin/bash
```

Then identify your device using `lsusb` like thi
```
[root@virtctl-usbredir ~]# lsusb
Bus 002 Device 002: ID 8087:8002 Intel Corp. 8 channel internal hub
...snip...
Bus 005 Device 003: ID 067b:2303 Prolific Technology, Inc. PL2303 Serial Port / Mobile Action MA-8910P
Bus 005 Device 004: ID 067b:2303 Prolific Technology, Inc. PL2303 Serial Port / Mobile Action MA-8910P
...snip...
Bus 005 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub
```

I can see that the USB device I want to redirect has a vendor:product ID of `067b:2303`. The only thing left to do is redirect the device like this

```
[root@virtctl-usbredir ~]# virtctl usbredir 067b:2303 rhel8-happy-golucky &
{"component":"","level":"info","msg":"port_arg: 'localhost:41949'","pos":"usbredir.go:157","timestamp":"2022-12-03T10:22:39.583643Z"}
{"component":"","level":"info","msg":"args: '[--device 067b:2303 --to localhost:41949]'","pos":"usbredir.go:158","timestamp":"2022-12-03T10:22:39.583747Z"}
{"component":"","level":"info","msg":"Executing commandline: 'usbredirect [--device 067b:2303 --to localhost:41949]'","pos":"usbredir.go:159","timestamp":"2022-12-03T10:22:39.584022Z"}
{"component":"","level":"info","msg":"Connected to usbredirect at 303.988732ms","pos":"usbredir.go:126","timestamp":"2022-12-03T10:22:39.887600Z"}
```

# Conclusion & Acknowledgements
:::success
Success! If you were watching your VM's kernel logs or your Windows VMs console when you ran the `virtctl usbredir ...` command you would have seen the Device Manager icon popup or the kernel log the discovery of a new device. There's room for improvement here, and I'll share a few more raw thoughts/notes below. I hope this helped you!

This all started with a customer request. While I was researching possible solutions I stumbled upon this [brief email exchange with Victor Toso](https://groups.google.com/g/kubevirt-dev/c/ZuAYp6twYp8/m/P-CfzR-0AgAJ) that gave me hope.
:::

# Issues

1. `usbredirect` crashes if you have duplicate `vendor:product` IDs (like my `lsusb` output shows above.) My customer has four identicle USB devices connected to their hypervisors.
2. I tried to redirect the same device from two 'virtctl usbredir' processes at the same time. The second process is blocked and reports an error - but the host saw a usb reset - so it could interfere with I/O
3. "redirect" performance was only 10MB/s when I tested reading a file (using `dd`) from a memory stick and sending it to `/dev/null` in a VM. I think "passthrough" would give much better (native?) performance.
4. The `Pod` and your `VirtualMachine` have to run in the same `namespace`/`project`. `usbredirect` appears to be working, but the VM never sees a new device attachment. :slightly_frowning_face:
5. I need to build some logic to automate the connection and locking to avoid trying to redirect the same device multiple times.
6. It would be nice to have a GUI / integration with the OpenShift Web Console. The PCI passthrough stuff is awesome! Let's extend that to USB devices!
    

# ==BELOW HERE ARE MY RAW THOUGHTS/NOTES - VIEWER DISCRETION IS ADVISED!==

## Advanced deployment options
If I continue with this USB redirect method I will have to figure out how to automate the connection process.

### DaemonSet/Deployment/StatefulSet

If we used a DaemonSet we may need some sort of locking mechanism to ensure that we are only redirecting usb exactly once among all the pods. This can be done simply with ConfigMaps which is how multiple controllers in Kubernetes maintain single-node leadership.

If we used a Deployment or a StatefulSet we would have slightly better control over how many pods are running at once.

### Container Command/Args

Whatever way we choose to run the Pod we will need to have some logic in the usbredirect container to:

- handle any sort of locking, if required
- start and manage the virtctl invocation

What is nice though is that this script can just be baked into the container image. We can pass through the USB vendor/product ID through env vars.



## Let OpenShift build our container image
My OpenShift cluster doesn't seem to grant access to the necessary RHEL8 repos, but if it did, I could use something like this to build and publish the container image.

```
---
kind: BuildConfig
apiVersion: build.openshift.io/v1
metadata:
  name: virtctl-usbredir
  namespace: jcall
spec:
  mountTrustedCA: true
  strategy:
    type: Docker
  source:
    dockerfile: |
      FROM registry.access.redhat.com/ubi8 AS builder
      RUN dnf download --source usbredir
      RUN dnf download usbutils
      RUN dnf -y builddep ./usbredir-* --enablerepo=codeready-builder-for-rhel-8-x86_64-rpms 
      RUN dnf -y install rpm-build gcc gcc-c++
      RUN rpm -ihv ./usbredir*.src.rpm && \
          rm ./usbredir*.src.rpm
      RUN rpmbuild -ba /root/rpmbuild/SPECS/usbredir.spec 
      RUN ln -s /root/rpmbuild/RPMS/x86_64/usbredir-[0-9]* / && \
          ln -s /root/rpmbuild/RPMS/x86_64/usbredir-server-[0-9]* /

      FROM registry.access.redhat.com/ubi8
      WORKDIR /root
      COPY --from=builder /*.rpm .
      RUN dnf -y install *.rpm && \
          dnf clean all && \
          rm -v *.rpm
      COPY ./oc /usr/local/bin
      COPY ./virtctl /usr/local/bin
      RUN  mkdir -p /etc/udev
      COPY ./hwdb.bin /etc/udev/
      COPY ./Containerfile-rpmbuild-usbredir /root/Containerfile
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