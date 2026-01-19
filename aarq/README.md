# Application Aware Quotas vs. Standard Quotas
This demo illustrates the limitations of standard Kubernetes **ResourceQuotas** when running Virtual Machines (VMs) and how **Application Aware Quotas** (AAQ or AARQ) solve these issues by understanding the VM lifecycle (e.g., Live Migration) and separating VM resources from container overhead.

**Prerequisites:**
* **Cluster:** OpenShift Container Platform **4.18+** (Tested on 4.20.8) with OpenShift Virtualization installed.
* **CLI Tools:** `oc` and `virtctl` installed. **Alternative:** You can replicate these steps using the OpenShift Web Console.

## Resource Quota
1 - Create a new project: 

```bash
oc new-project rq-demo
```

2 - Create the ResourceQuota: 

```bash
oc create -f resource-quota.yaml
```

3 - Check the created resource status: 

```bash
oc describe resourcequota/example-resource-quota
```

4 - Create VM Pool and scale it to 6. Each VM requires 1 vCPU and 1Gi of RAM:

```bash
oc apply -k git@github.com:openshift-examples/kustomize/components/vmpool-no-load
oc scale vmpool/no-load --replicas 6
```

5 - Observation: 
  - Run `oc get vmi`.
  - **Result**: You will likely see **3 Running** and **3 Starting/Pending**.
  - **Reason**: You hit the **Memory Limit** defined in the ResourceQuota.
    - **The Conflict:** Your 3 running VMs are already using ~7.6 Gi of the 10 Gi limit. The 4th VM requests ~2.5 Gi of limit capacity (due to Pod overhead + overcommit settings), pushing the total to ~10.1 Gi, which exceeds the quota.
  - **Observation on CPU Overcommit:** `requests.cpu` is only 318m (approx 0.3 CPU), even though you have 3 vCPUs running. OpenShift Virtualization applies a CPU Overcommit by default (controlled by `cpuAllocationRatio` in the HyperConverged operator).

6 - Choose a *Running* VM and migrate it.
```bash
virtctl migrate $(oc get vmi | awk '{ if ($3 == "Running") { print $1; exit}}')
# Get events to see errors
oc get events --sort-by=.metadata.creationTimestamp
```
  - **Result**: The migration does not start. 
  - **Why**: To Live Migrate a VM, OpenShift must create a second Launcher Pod on the target node before stopping the source Pod.
    - `Current Usage (3 VMs) + Target Pod (1 VM) > Quota Limit`.
    - The standard ResourceQuota rejects the new Pod, effectively locking you out of maintenance operations.

## Application Aware Resource Quota
1 - **Enable AAQ Operator**. We must update the HyperConverged CR:
```bash
oc patch hco kubevirt-hyperconverged -n openshift-cnv \
 --type json -p '[{"op": "add", "path": "/spec/enableApplicationAwareQuota", "value": true}]'
``` 
2 - Configure AAQ operator:
```bash
oc patch hco kubevirt-hyperconverged -n openshift-cnv --type merge -p '{
  "spec": {
    "applicationAwareConfig": {
      "vmiCalcConfigName": "DedicatedVirtualResources",
      "namespaceSelector": {
        "matchLabels": {
          "app": "aarq-operator-demo"
        }
      },
      "allowApplicationAwareClusterResourceQuota": true
    }
  }
}'
```
- `vmiCalcConfigName` specify how to measure the *size* of a Virtual Machine. 
Possible options are:
    - `VmiPodUsage`: Counts the total resource footprint of the Pod running the VM (Guest VM specs + Kubernetes system overhead). VMs share the same resource limits (`requests.cpu`) as standard containers.
    - `VirtualResources`: Counts only the defined Guest VM specifications (e.g., 4GB RAM), ignoring the pod overhead. VMs still share the same resource limits (`requests.cpu`) as standard containers, but the calculation is based purely on the VM definition.
    - `DedicatedVirtualResources`: Counts only the defined Guest VM specifications but segregates them into a distinct budget. Resources are tracked using a special suffix (e.g., `requests.cpu/vmi`), ensuring that Virtual Machines do not compete with standard containers for the same quota.
- `namespaceSelector`: Determines the namespaces for which an AAQ scheduling gate is added to pods when they are created. If a namespace selector is not defined, the AAQ Operator targets namespaces with the `application-aware-quota/enable-gating` label as default.

3 - Create a new project and label it:
```bash
oc new-project aarq-demo
oc label namespace aarq-demo app=aarq-operator-demo
```
4 - Create the Application Aware Resource Quota:

```bash
oc create -f application-aware-resource-quota.yaml
```

5 - Check the created resource: 
```bash
oc describe arq example-application-aware-resource-quota -n aarq-demo
```
6 - Create VM Pool and scale it to 6:

```bash
oc apply -k git@github.com:openshift-examples/kustomize/components/vmpool-no-load
oc scale vmpool/no-load --replicas 6
```
7 - Check Application Aware Resource Quota again: 

```bash 
oc describe arq example-application-aware-resource-quota -n aarq-demo
```
- **Observation**: you should see that `requests.cpu/vmi` is fully utilized (3/3).

8 - Check VMI status `oc get vmi`; 
  - **Result**: 3 VMs are **Running** and 3 are in **Scheduling (Gated)** state.
  - **Reason**: Unlike Part 1, you are now stopped by **vCPU Requests**, not Memory Limits. AARQ checks Guest VM specifications, not pod launcher requests.  

9 - Choose a *Running* VM and migrate it.
```bash
virtctl migrate $(oc get vmi | awk '{ if ($3 == "Running") { print $1; exit}}')
# Watch the migration
watch oc get vmi
```
  - **Result**: The migration **Succeeds**.
  - **Why**: Even though `3 Running + 1 Migration Target = 4 vCPUs` (exceeding the limit of 3), AAQ identifies the migration intent and allows the temporary burst.