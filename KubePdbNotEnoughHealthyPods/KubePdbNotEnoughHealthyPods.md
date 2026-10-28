# KubePdbNotEnoughHealthyPods

**PrometheusRule Source:** `kube-state-metrics` · **Pending For:** `15m` · **Severity:** `Warning`

## Meaning

This alert indicates that a PodDisruptionBudget (PDB) has fewer healthy pods than its configured requirement.

The condition may restrict voluntary pod evictions and delay node maintenance.

## Impact

- **Upgrade Delays:** Node drains may be delayed during cluster upgrades or MachineConfig updates.
- **Maintenance Restrictions:** Voluntary pod evictions may be denied to protect workload availability.
- **Reduced Availability:** Fewer healthy replicas may increase the risk of service disruption.

## Diagnosis

### 1. Identify the Affected PDB

Run the following query in the Prometheus console or OpenShift Metrics UI:

```promql
ALERTS{
  alertname="KubePdbNotEnoughHealthyPods",
  alertstate="firing"
}
```

Identify the affected namespace and PDB from the returned labels, where available.

List PDBs and their health status:

```bash
oc get pdb -A \
  -o custom-columns='NAMESPACE:.metadata.namespace,NAME:.metadata.name,CURRENT:.status.currentHealthy,DESIRED:.status.desiredHealthy,ALLOWED:.status.disruptionsAllowed'
```

Inspect the affected PDB:

```bash
oc get pdb <pdb-name> -n <namespace> -o yaml
```

Review `currentHealthy`, `desiredHealthy`, `disruptionsAllowed`, and the PDB selector.

### 2. Check Upgrade and Node Status

```bash
oc get clusterversion
oc get mcp
oc get nodes
```

Check for:

- Cluster upgrades in progress.
- MachineConfigPools with `UPDATING=True` or `DEGRADED=True`.
- Nodes in `NotReady` or `SchedulingDisabled` status.
- Delayed node reboots or incomplete maintenance operations.

### 3. Check Cluster Operators

```bash
oc get co
```

Identify operators reporting `Available=False`, `Progressing=True`, or `Degraded=True`.

Inspect the relevant operator:

```bash
oc describe clusteroperator <operator-name>
```

Review operator conditions, events, and logs for reconciliation failures.

### 4. Check the Affected Pods

Inspect the PDB configuration:

```bash
oc get pdb <pdb-name> -n <namespace> -o yaml
```

Use the configured selector to identify the associated pods. For example, if the selector contains `app=my-app`:

```bash
oc get pods -n <namespace> -l app=my-app -o wide
```

Check for pods in `Pending`, `CrashLoopBackOff`, or `Error` states, and pods that are not Ready.

Inspect pod events and logs:

```bash
oc describe pod <pod-name> -n <namespace>
oc logs <pod-name> -n <namespace> --all-containers=true
```

For restarted containers, inspect the previous logs:

```bash
oc logs <pod-name> -n <namespace> \
  --all-containers=true --previous
```

## Mitigation

| Scenario | Action / Mitigation | Verification |
|---|---|---|
| **Upgrade or node maintenance in progress** | Monitor node and MachineConfigPool status. Investigate stalled drains, node reboots, or degraded pools. Avoid bypassing PDB protection without following a supported procedure. | Check `oc get nodes` and `oc get mcp`. Confirm that the maintenance operation progresses normally. |
| **Pods Pending** | Check CPU and memory availability, node selectors, affinity rules, taints, and tolerations. Add cluster capacity if required and supported. | Confirm that affected pods are scheduled and become Ready. |
| **Pods failing** | Review container logs, pod events, readiness probes, and storage or networking dependencies. Restore the workload using the supported recovery procedure. | Confirm that affected pods become Ready and healthy replica counts recover. |
| **Cluster Operator degraded** | Inspect operator conditions, events, and logs. Resolve the underlying reconciliation failure using the relevant Red Hat troubleshooting documentation. | Verify that the operator is healthy and the managed workload has recovered. |
| **No apparent cause** | Review the PDB selector, status, alert expression, and metric labels. Confirm that the affected pods match the PDB selector. | Verify that the PDB health requirement is satisfied and the alert condition is resolved. |

**Note:** Do not manually weaken platform PDBs or modify operator-managed resources without an approved, supported procedure.

## Verification

Run the following commands after mitigation:

```bash
oc get pdb <pdb-name> -n <namespace>
oc get pods -n <namespace>
oc get co
```

Verify that:

- The healthy pod count meets the PDB requirement.
- `disruptionsAllowed` reflects the expected state.
- The affected Cluster Operator is healthy, where applicable.
- The alert is no longer firing in Prometheus.

Confirm that the alert remains resolved for the configured `15m` pending period before closing the incident.

## References

- [Kubernetes: Configure a PodDisruptionBudget](https://kubernetes.io/docs/tasks/run-application/configure-pdb/)
- [OpenShift Container Platform Documentation](https://docs.openshift.com/container-platform/latest/)
