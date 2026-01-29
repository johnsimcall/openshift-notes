# High Availability for OpenShift workloads

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
