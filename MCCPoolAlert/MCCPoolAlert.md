# MCCPoolAlert

**PrometheusRule Source:** `` · **Alert Severity:** `Warning` · **Pending Period:** `0m`. **[Runbook](https://github.com/openshift/runbooks/blob/master/alerts/machine-config-operator/MachineConfigControllerPoolAlert.md)**

This alert is triggered when the Machine Config Controller (MCC) detects that one or more nodes have overlapping labels matching both the `master` pool and a custom pool.

When a node belongs to both pools, a choice must be made as to which configuration is honored. In this scenario, the MCC defaults to honoring only the `master` pool.

****

**Expression:**

```bash
mcc_pool_alert > 0
```

## Impact

When nodes have overlapping labels for both the master and custom pools:

- The Machine Config Controller defaults to applying only the `master` pool configuration.
- Custom machine configurations tied to the custom pool will **not** be applied to the affected nodes.

## Diagnosis

### 1. Identify the Machine Config Controller pod

```bash
oc get pods -n openshift-machine-config-operator \
  -l k8s-app=machine-config-controller
```

### 2. Set the controller pod name

Set the `CONTROLLERPOD` environment variable using the pod name retrieved in the previous step:

```bash
export CONTROLLERPOD="<your-machine-config-controller-pod-name>"
```

### 3. Check the controller logs

```bash
oc -n openshift-machine-config-operator \
  logs -f "$CONTROLLERPOD" \
  -c machine-config-controller
```

### 4. Verify the pool overlap

Look for a log entry similar to the following, which confirms that a node matches a custom pool selector and the controller defaults to the master pool:

```text
Found xxxxxx node that matches selector for custom pool yyyyy, defaulting to xxxxx. This node will not have any custom role configuration as a result. Please review the node to make sure this is intended
```

## Mitigation

Setting up a node with both a master and custom MachineConfigPool (MCP) is supported, but it will unconditionally activate this alert. Any MachineConfigs (MCs) matching the custom pool will be ignored on nodes that also match the master pool.

If this behavior is **not intentional**, follow these steps to correct the node selectors.

### 1. Inspect the custom MachineConfigPool

```bash
oc get mcp <custom-pool> -o yaml
```

Review the `spec` and identify the node selector and labels that cause the overlap.

### 2. Edit the custom MachineConfigPool

```bash
oc edit mcp <custom-pool>
```

MachineConfigPools apply configurations to nodes that match the `spec.nodeSelector.matchLabels` configuration. Adjust the selector to ensure that only the intended nodes match the custom pool.

Use the following structure as a reference:

```yaml
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfigPool
metadata:
  name: <custom-pool>
spec:
  machineConfigSelector:
    matchExpressions:
      - key: machineconfiguration.openshift.io/role
        operator: In
        values:
          - <role-value-1>
          - <role-value-2>
  maxUnavailable: null
  nodeSelector:
    matchLabels:
      <node-label-key>: <node-label-value>
  paused: false
```

### 3. Verify the correction

Check the MCP status and confirm that only the intended nodes match the custom pool:

```bash
oc get mcp
oc get nodes --show-labels
```

Query the alert metric in Prometheus:

```promql
mcc_pool_alert
```

Confirm that `MCCPoolAlert` resolves when the overlapping pool condition is no longer present.
