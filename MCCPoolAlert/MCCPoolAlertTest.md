# Reproduce MCCPoolAlert

## Prerequisites

- Access to an OpenShift test cluster with `cluster-admin` privileges.
- A disposable test cluster or an approved maintenance window, because overlapping MachineConfigPool selectors can affect node configuration management.


## Reproduction Steps

### 1. Add the custom role label to a control-plane node

```bash
oc label node <control-plane-node> \
  node-role.kubernetes.io/mccpooltest=""
```

### 3. Create the custom MachineConfigPool

Create `mccpooltest-mcp.yaml` with the following configuration:

```yaml
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfigPool
metadata:
  name: mccpooltest
spec:
  machineConfigSelector:
    matchExpressions:
      - key: machineconfiguration.openshift.io/role
        operator: In
        values:
          - worker
          - mccpooltest
  nodeSelector:
    matchLabels:
      node-role.kubernetes.io/master: ""
      node-role.kubernetes.io/mccpooltest: ""
```

### 4. Apply the custom MachineConfigPool

```bash
oc apply -f mccpooltest-mcp.yaml
```

### 5. Verify the alert

Query the metric in Prometheus:

```promql
mcc_pool_alert
```

**Expected result:** The metric value becomes greater than `0` for the affected node.

Verify that the alert is firing:

```promql
ALERTS{alertname="MCCPoolAlert",alertstate="firing"}
```

### 6. Verify Machine Config Controller logs

```bash
oc logs -n openshift-machine-config-operator \
  <machine-config-controller-pod> \
  -c machine-config-controller \
  --since=10m
```

Inspect the logs for messages indicating that the node matches a custom pool and the controller defaults to the master pool.

## Cleanup

### 1. Delete the custom MachineConfigPool

```bash
oc delete mcp mccpooltest
```

### 2. Remove the test role label from the control-plane node

```bash
oc label node <control-plane-node> \
  node-role.kubernetes.io/mccpooltest-
```

### 3. Verify cluster recovery

```bash
oc get mcp
oc get nodes --show-labels
```

Confirm that the MachineConfigPools return to their expected state, `mcc_pool_alert` returns to `0`, and `MCCPoolAlert` resolves.

> **Caution:** Perform this test only on a disposable test cluster or in an approved maintenance window. Verify the original MCP configuration before cleanup and restore it if it differs from the test configuration.
