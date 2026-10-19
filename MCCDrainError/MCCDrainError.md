# MCCDrainError

**PrometheusRule Source:** `machine-config-operator`  . **Pending For:**-   . **Severity:** Warning . [Runbook](https://github.com/openshift/runbooks/blob/master/alerts/machine-config-operator/MachineConfigControllerDrainError.md)

This alert indicates that the Machine Config Controller (MCC), part of the Machine Config Operator (MCO), failed to drain a node. This can prevent the MCO from rebooting the node to apply machine configuration changes or progressing with an ongoing cluster upgrade.

The alert fires immediately when the `mcc_drain_err` metric is greater than zero.

### Drain Retry and Timeout Progression

When the MCC attempts to update a node, it cordons the node and begins evicting workloads. The controller follows an internal retry and timeout cycle:

- **At 1 minute 30 seconds:** If a pod cannot be evicted or terminated within the drain timeout, the controller records a failure, waits 1 minute, and retries.
- **After 10 minutes:** If drain failures continue, the retry backoff increases to 5 minutes.
- **After 1 hour:** The drain operation reaches its hard timeout. The node is marked as Degraded in the MachineConfigPool (MCP), and updates to that pool are blocked until the issue is resolved.


## Diagnosis

Node drain failures are typically caused by the following issues:

| Type | Meaning | What to Investigate |
|---|---|---|
| `pdb-violation` | A PodDisruptionBudget (PDB) prevents eviction of remaining replicas. | PDBs targeting pods on the affected node, especially those with zero allowed disruptions. |
| `stuck-finalizer` | A pod remains in Terminating state because a finalizer has not been cleared. | Pod metadata finalizers and the responsible controller or operator logs. |
| `storage-unmount` | The kubelet or CSI driver cannot unmount persistent volumes. | Node storage mounts, CSI driver pods, and remote storage connectivity, such as NFS, SAN, or cloud disks. |
| `failing-webhook` | An admission webhook rejects or times out during an API operation. | API server logs, webhook configuration, and webhook service availability. |

### 1. Identify the Affected Node and Controller Pod

Check the `exported_node` alert label, if present, to identify the failing node. Inspect the MachineConfigPools and locate the Machine Config Controller pod.

```bash
# Inspect MachineConfigPools
oc get mcp

# Inspect nodes and their current status
oc get nodes

# Capture the active Machine Config Controller pod
CONTROLLER_POD=$(oc get pods -n openshift-machine-config-operator \
  -l k8s-app=machine-config-controller \
  -o jsonpath='{.items[0].metadata.name}')

echo "$CONTROLLER_POD"
```

### 2. Inspect Machine Config Controller Logs

Examine the MCC logs to identify the node and pod blocking the drain.

```bash
oc -n openshift-machine-config-operator logs "$CONTROLLER_POD" \
  -c machine-config-controller --tail=500 | grep -E "drain|evict"
```

Look for log messages indicating the following conditions:

**Drain initiation**

```text
node <node-name>: initiating drain
```

**Pod eviction progress**

```text
node <node-name>: Evicted pod <namespace>/<pod-name>
```

**Drain retries before 10 minutes**

```text
Drain failed. Waiting 1 minute then retrying.
```

**Sustained drain retries after 10 minutes**

```text
Drain has been failing for more than 10 minutes. Waiting 5 minutes then retrying.
```

**Hard timeout after 1 hour**

```text
drain exceeded timeout: 1h0m0s. Will continue to retry.
```

The controller may also report:

```text
Degraded Machine: <node-name> and Degraded Reason: failed to drain node: <node-name> after 1 hour.
```

Exact log wording can vary by OpenShift release. Use the surrounding log entries to identify the underlying error.

### 3. Diagnose the Blocking Resource

#### i. If Caused by a PodDisruptionBudget (PDB)

List the pods on the affected node and inspect the PDBs.

```bash
# List pods scheduled on the affected node
oc get pods -A --field-selector spec.nodeName=<node-name>

# Inspect PodDisruptionBudgets across namespaces
oc get pdb -A
```

Inspect the relevant PDB's `ALLOWED DISRUPTIONS`, `MIN AVAILABLE`, and `MAX UNAVAILABLE` values.

```bash
oc describe pdb <pdb-name> -n <namespace>
```

 #### ii.  If Caused by Stuck Finalizers or Terminating Pods

Identify pods that are stuck terminating and inspect their finalizers.

```bash
# List pods on the affected node
oc get pods -A --field-selector spec.nodeName=<node-name>

# Inspect finalizers on the affected pod
oc get pod <pod-name> -n <namespace> \
  -o jsonpath='{.metadata.finalizers}{"\n"}'

# Inspect pod details and events
oc describe pod <pod-name> -n <namespace>
```

Determine why the responsible controller has not completed deletion before modifying finalizers.

#### iii. If Caused by Storage Detach or Unmount Errors

Inspect pod events, node events, and storage-related errors.

```bash
oc describe pod <pod-name> -n <namespace>

oc describe node <node-name>
```

Check the CSI driver and storage provider logs where applicable. Look for volume attachment failures, mount errors, and connectivity problems.

#### iv. If Caused by Admission Webhooks

Inspect API server logs for webhook failures and timeouts.

```bash
oc logs -n openshift-kube-apiserver \
  -l app=kube-apiserver \
  -c kube-apiserver --tail=200 |
  grep -E "failed calling webhook|context canceled"
```

List the configured admission webhooks.

```bash
oc get validatingwebhookconfigurations
oc get mutatingwebhookconfigurations
```

Identify the failing webhook and verify its service availability, endpoints, TLS configuration, and timeout settings.

## Mitigation

### 1. Mitigate PodDisruptionBudget Constraints

If a PDB is blocking eviction, determine whether the application can safely tolerate the disruption. Prefer restoring application availability or scaling replicas to satisfy the existing PDB.

If a temporary PDB change is necessary, obtain application-owner approval and record the original configuration before modifying it.

For example, if the PDB uses `minAvailable`, a temporary change may be required:

```bash
oc patch pdb <pdb-name> -n <namespace> \
  --type=merge -p '{"spec":{"minAvailable":0}}'
```

**Caution:** This removes the configured minimum availability requirement for the affected workload. Use only when the operational risk is understood, and restore the original value after the drain and node update complete.

### 2. Resolve Stuck Finalizers or Terminating Pods

Determine which controller owns the finalizer and why cleanup has not completed. Correct the underlying issue and allow normal deletion to finish.

Only consider force deletion when the consequences are understood and the workload owner approves.

```bash
oc delete pod <pod-name> -n <namespace> \
  --force --grace-period=0
```

**Caution:** Force deletion does not guarantee that the process has stopped on the node or that storage cleanup has completed. Do not remove finalizers indiscriminately, particularly for stateful workloads.

### 3. Resolve Storage Detach or Unmount Errors

- Restore connectivity to the storage backend.
- Resolve CSI driver or volume attachment errors.
- Verify that volumes can be unmounted safely.
- Review storage-provider guidance before manually detaching volumes or deleting attachment resources.

Retry the node update only after the underlying storage issue is resolved.

### 4. Resolve Admission Webhook Failures

Correct the webhook service availability, endpoints, TLS configuration, or timeout issues.

If a webhook must be temporarily disabled, follow the webhook owner's and operator's supported procedure. Back up the configuration and assess the security and admission-policy implications before making changes.

Avoid deleting an entire `ValidatingWebhookConfiguration` or `MutatingWebhookConfiguration` without confirming that the change is safe and supported. Restore and verify the intended configuration immediately after the recovery operation.

## Verification

### 1. Monitor Machine Config Controller Logs

```bash
oc -n openshift-machine-config-operator logs "$CONTROLLER_POD" \
  -c machine-config-controller -f
```

Verify that the drain operation progresses and that the controller no longer reports repeated drain failures.

### 2. Verify MachineConfigPool Status

```bash
oc get mcp
oc describe mcp <pool-name>
```

Confirm that the pool is no longer degraded and eventually reaches `Updated=True`, with `Updating=False` and `Degraded=False`, as applicable to the operation.

### 3. Verify Node Status

```bash
oc get nodes
oc get node <node-name>
```

Confirm that the node is Ready and that any intended configuration update has completed.

### 4. Confirm Alert Resolution

The alert should clear automatically when `mcc_drain_err` returns to zero and the alert expression no longer evaluates to true. Confirm the alert state in the OpenShift monitoring interface.
