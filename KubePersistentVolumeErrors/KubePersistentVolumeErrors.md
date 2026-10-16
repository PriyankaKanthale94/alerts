# KubePersistentVolumeErrors

**PrometheusRule Source:** `cluster-monitoring-operator` · **Pending For:** `5m` · **Severity:** `Warning` · **Runbook:** KubePersistentVolumeErrors

---

## Meaning

This alert indicates that a PersistentVolume (PV) in the monitored namespace scope has remained in the `Failed` or `Pending` phase for at least 5 minutes.

- **Failed:** The PV has encountered a failure. Inspect its events, reclaim policy and storage-provider status to determine the cause.
- **Pending:** The PV has not reached the `Available` or `Bound` phase. Investigate provisioning, binding constraints and storage-controller errors.

PersistentVolumes are cluster-scoped resources. The `namespace` label used by the alert expression is a metric label and does not necessarily identify the namespace of the PVC referenced by the PV.

## Expression

```promql
kube_persistentvolume_status_phase{
  job="kube-state-metrics",
  namespace=~"(openshift-.*|kube-.*|default)",
  phase=~"Failed|Pending"
} > 0
```

Confirm the expression and labels against the actual firing alert in the cluster.

## Impact

- Workloads that require affected storage may remain pending, fail to start or experience storage-related errors.
- Failed provisioning or volume attachment can disrupt applications and platform services.
- Volumes that cannot be reclaimed may leave underlying storage resources allocated.
- Repeated provisioning failures may prevent applications and platform components from obtaining required storage.

## Diagnosis

### 1. Identify the affected PVs

Inspect the firing alert and its labels in the OpenShift web console under **Observe → Alerting → Alerts**.

List PVs and their phases:

```bash
oc get pv
```

Inspect the PV status metric in the Prometheus console, using the exact expression from the alert. Correlate the metric labels with the PV names and phases reported by Kubernetes.

or 
```bash
oc get pv -o json | jq -r '
  .items[] | 
  select(.status.phase == "Failed" or .status.phase == "Pending") | 
  select(.spec.claimRef.namespace | test("^(openshift-.*|kube-.*|default)$")) | 
  [.metadata.name, .status.phase, .spec.claimRef.namespace, .spec.claimRef.name, .spec.storageClassName] | 
  @tsv'
```
### 2. Inspect the affected PV

```bash
oc describe pv <pv-name>
oc get pv <pv-name> -o yaml
```

Check:

- Current phase and status conditions.
- PV events and failure messages.
- `persistentVolumeReclaimPolicy`.
- `storageClassName` and CSI driver.
- `claimRef`, if present.
- Volume handle and any deletion or protection finalizers.

### 3. Inspect the associated PVC, if present

Use the namespace and name in `spec.claimRef`:

```bash
oc -n <claim-namespace> get pvc <claim-name>
oc -n <claim-namespace> describe pvc <claim-name>
```

If the PVC does not exist, determine whether it was intentionally deleted, whether the PV is being reclaimed, or whether a stale claim reference remains.

### 4. Check the StorageClass and provisioner

```bash
oc get storageclass
oc get storageclass <storage-class-name> -o yaml
```

Verify the provisioner, volume binding mode, reclaim policy and relevant topology or capacity constraints.

### 5. Check storage operator health

```bash
oc get clusteroperator storage
oc describe clusteroperator storage
```

If a storage operator is degraded, inspect its conditions and events to identify the affected operand.

### 6. Inspect the relevant CSI driver or storage backend

Discover the CSI drivers:

```bash
oc get csidrivers
```

Identify the controller and node pods managed by the relevant storage operator. Inspect their logs and events for provisioning, deletion, attachment, authorization or backend errors.

For OpenShift Data Foundation, inspect the health of the ODF and Ceph components using the tools and procedures supported by the installed version.

## Mitigation

Apply the action appropriate to the confirmed failure. Do not delete a PV, clear finalizers, or modify the underlying storage until ownership, data-retention requirements and workload dependencies have been verified.

| Scenario | What to check | Action / Mitigation |
|---|---|---|
| PV is `Failed` following a deletion or reclamation error | PV events, reclaim policy, CSI controller logs and backend deletion errors | Resolve the underlying backend or CSI issue. Confirm whether the volume contains data that must be retained before retrying cleanup. |
| PV is `Pending` because provisioning failed | PVC events, StorageClass provisioner, CSI controller logs and backend quotas | Correct the reported provisioning, capacity, topology or authorization issue. Allow the provisioner to retry and verify the resulting PV state. |
| PV is `Pending` with a stale claim reference | PV `spec.claimRef`, referenced PVC and PVC UID | For a manually managed static PV, clear the claim reference only after confirming that the previous claim is no longer using the volume and that reclaiming it is safe. |
| Storage operator or CSI controller is degraded | ClusterOperator conditions, pod events and controller logs | Resolve the reported operator, scheduling, image, credential or connectivity problem using the procedure for the affected storage driver. |
| Storage backend is unhealthy or out of capacity | Backend health, pool capacity, quotas and provider events | Restore backend health or available capacity. For ODF, follow the supported Ceph/ODF recovery procedure before retrying volume operations. |
| PV remains in `Terminating` | Active PVCs and pods, PV protection finalizers, CSI cleanup errors | Resolve outstanding consumers or storage cleanup failures. Remove a finalizer only when the relevant controller's cleanup requirements have been satisfied and the action is supported. |

### If the cause remains unknown

Collect the following information:

```bash
oc describe pv <pv-name>
oc get pv <pv-name> -o yaml
oc get events -A --sort-by=.lastTimestamp
oc get clusteroperator storage
oc get csidrivers
```

Collect relevant CSI controller logs and an OpenShift must-gather bundle according to the storage provider's support procedure. Escalate to the storage infrastructure team or Red Hat Support when required.
