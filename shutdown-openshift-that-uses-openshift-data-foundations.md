# Shutting down OpenShift with Ceph / OpenShift Data Foundations (simplified)

I created a simple `bash` script to help me shutdown my hyperconverged 3-node OpenShift Container Platform (OCP) cluster running Virtual Machines with OpenShift Data Foundation (ODF.) I need this script because following the [OCP shutdown instructions](https://docs.openshift.com/container-platform/latest/backup_and_restore/graceful-cluster-shutdown.html) results in a hung / stalled system. The hang occurs because of a race condition between shutting down ODF and the applications using ODF. My simplified script orders the shutdown of VMs & Pods before finally shutting down ODF & OCP.

:::info
**Update:** Big thanks to Jason Kincl who provided the `json` queries to discover ALL workloads using ODF storage. His approach is more efficient than the individual shutdown sections I used previously, and which didn't account for additional workloads using ODF.
:::

The basic workflow is:
1. Setup some connection variables
2. Mark the OpenShift nodes as unschedulable which prevents Pods & VMs from restarting
3. Stop everything using ODF
    a. VMs - gracefully shutdown with `oc delete VirtualMachineInstance/...`
    b. Monitoring - graceful shutdown with `oc delete Pod/...`
    c. other apps
6. Shutdown the nodes (remaining ODF processes, kube-apiserver, etcd, and other OCP processes)

Shutdown ODF first - https://access.redhat.com/solutions/6642401
Shutdown OCP last - https://docs.openshift.com/container-platform/latest/backup_and_restore/graceful-cluster-shutdown.html

The script below tries to gracefully shutdown a 3-node / "compact" OCP cluster. Some coordination is required to shut down cleanly when ODF, Logging, and Virtualization are deployed together.

```bash=
#!/bin/bash
CLUSTER_NAME="tacos"
OC_BIN=/usr/local/bin/oc
KUBECONFIG=/home/jcall/ocp-tacos/kubeconfig
OC_CMD="$OC_BIN --kubeconfig=$KUBECONFIG"


# check if we're connected to the correct cluster
CONNECTED_CLUSTER=$($OC_CMD whoami --show-console)
if [[ $CONNECTED_CLUSTER =~ $CLUSTER_NAME ]]; then
  CERT_EXPIRE=$($OC_CMD -n openshift-kube-apiserver-operator get secret/kube-apiserver-to-kubelet-signer -o jsonpath='{.metadata.annotations.auth\.openshift\.io/certificate-not-after}')
  echo "Please restart the cluster before the certificates expire: $(date -d $CERT_EXPIRE)"
else
  echo "Error: Not connected to $CLUSTER_NAME"
  echo "found $CONNECTED_CLUSTER instead!"
  exit 1
fi


echo "Marking all nodes as Unschedulable"
for i in $($OC_CMD get nodes -o name); do
  $OC_CMD adm cordon $i
done


echo ; echo "Find all Pods and VMs using ODF storage"
export STORAGECLASSES=$(oc get sc -o json | jq -j '.items | .[] | select(.provisioner|endswith("csi.ceph.com")) | .metadata.name')
export PVCS=$(oc get pvc -A -o json | jq -j --arg sc "$STORAGECLASSES" '.items | .[] | select(.spec.storageClassName | inside($sc) )| .metadata.name')
export PODS=$(oc get pods -A -o json | jq -r --arg pvc "$PVCS" '.items | .[] | select(.spec.volumes | .[] | try(.persistentVolumeClaim) | select(.claimName | inside($pvc))) | .metadata.namespace+" "+.metadata.name' | sort | uniq | tee /dev/tty)

echo ; echo "Enter to continue to delete the above pods, or press Ctrl + C to abort"
read yes

echo ; echo "Sending graceful shutdown command to Pods and VMs"
echo "$PODS" | while read line; do
  oc delete pod -n $line
done


# nothing should be using ODF at this point, move on to shutting down the nodes
echo ; echo "Telling the nodes to shutdown/halt in 5 minutes..."
for node in $($OC_CMD get nodes -o name); do
  oc debug $node -- chroot /host shutdown -h 5
done


echo ; echo "Please remember to \"uncordon\" the nodes when the cluster is restarted!"
echo "For example, using this loop command:"
echo "for i in $(oc get nodes -o name); do oc adm uncordon $i; done"
```
