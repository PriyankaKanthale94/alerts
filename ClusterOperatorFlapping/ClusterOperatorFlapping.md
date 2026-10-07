# ClusterOperatorFlapping

**PrometheusRule Source:** `Platform` · **Pending For:** `10m` · **Severity:** `Warning`

---
This alert indicates that Cluster operator up status is changing often. This behavior may cause cluster upgrades to become unstable.


**Expression:**

```promql
max by (namespace, name) (
  changes(cluster_operator_up{job="cluster-version-operator"}[2m]) > 2
)
```

The alert fires when the `cluster_operator_up` metric changes more than **2 times within 2 minutes** and the condition remains true for **10 minutes**.

## Impact

- Cluster upgrades may become unstable or fail to progress.
- The affected operator may repeatedly transition between healthy and unhealthy states.




## Diagnosis


## Variables

 `$labels.name` == `ClusterOperator`

| Check | Command | What to look for |
|---|---|---|
| Check upgrade status | `oc adm upgrade` | Check whether the affected operator is preventing or delaying an upgrade. |
| Check operator status | `oc get -o yaml clusteroperator <operator-name>` | Review `Available`, `Progressing`, and `Degraded` conditions, including `reason`, `message`, and `lastTransitionTime`, to identify the cause of the state changes. |
| Check ClusterVersion status | `oc get clusterversion version -o yaml` | Review upgrade conditions, progress, and any errors associated with the cluster upgrade. |
| Check operator pods | `oc get pods -A` | Identify the pods associated with the affected operator and check for restarts, `CrashLoopBackOff`, `Pending`, or unavailable pods. |
| Check operator logs | `oc logs -n <namespace> <pod-name> --since=1h` | Look for repeated errors or reconciliation failures corresponding to the state changes. |
| Check recent events | `oc get events -A --sort-by='.lastTimestamp'` | Look for events occurring when the operator state changes. |
| Check Cluster Version Operator logs | `oc logs -n openshift-cluster-version deploy/cluster-version-operator --since=1h` | Look for errors or repeated reconciliation activity related to the affected `ClusterOperator`. |
| Reason is unknown | `oc adm inspect clusteroperator/<operator-name>` | Collect debugging data for further analysis |

## Mitigation

| Cause / Error | What to check | Mitigation |
|---|---|---|
| **Operator repeatedly becomes `Degraded=True`** | `oc get -o yaml clusteroperator <operator-name>`; review `reason` and `message`. | Resolve the configuration, dependency, or resource issue reported by the condition. |
| **Operator reconciliation errors** | Operator logs for repeated reconciliation failures or errors. | Address the resource, configuration, permission, or dependency identified in the error. |
| **Dependent component is unhealthy** | Operator conditions and logs for references to another unhealthy component. | Resolve the dependent component issue and verify that the affected operator returns to a stable state. |
| **Certificate, authentication, or authorization failure** | Operator logs for TLS, certificate, authentication, or RBAC errors. | Correct the affected certificate, credentials, or RBAC configuration and verify operator recovery. |
| **Upgrade-related failure** | `oc adm upgrade` and `oc get clusterversion version -o yaml`. | Resolve the condition reported by the Cluster Version Operator before continuing the upgrade. |
## Verification

Confirm that the affected operator remains stable:

```bash
oc get clusteroperator <operator-name>
```

Verify that the alert condition is no longer met:

```promql
max by (namespace, name) (
  changes(cluster_operator_up{job="cluster-version-operator"}[2m]) > 2
)
```

The alert should clear after the flapping condition stops and the `10m` pending period has elapsed.
