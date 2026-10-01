# AlertmanagerConfigInconsistent

**PrometheusRule Source:** `` · **Pending For:** `20m` · **Severity:** `Warning` · **Runbook:** [AlertmanagerConfigInconsistent](https://github.com/prometheus-operator/runbooks/blob/main/content/runbooks/alertmanager/AlertmanagerConfigInconsistent.md?utm_source=chatgpt.com)

---

## Meaning

This alert indicates that Alertmanager instances within the same cluster are running with different configurations.

The alert compares the `alertmanager_config_hash` metric for Alertmanager instances belonging to the same service. The alert fires when more than one distinct configuration hash is detected.

**Alert Expression:**

```promql
count by (namespace, service) (
  count_values by (namespace, service) (
    "config_hash",
    alertmanager_config_hash{job=~"alertmanager-main|alertmanager-user-workload"}
  )
) != 1
```

## Impact

Configuration inconsistency can cause different Alertmanager instances to process the same alerts differently. Depending on the configuration difference, this can result in:

- Alerts being routed to different receivers.
- Notifications being duplicated or not sent as expected.
- Different silences or routing rules being applied by different instances.

## Diagnosis

### 1. Identify the Mismatched Configuration Hashes

Query the metric to identify the Alertmanager instances and their configuration hashes:

```promql
alertmanager_config_hash{job=~"alertmanager-main|alertmanager-user-workload"}
```

Identify which instances have different `config_hash` values.

### 2. Check Alertmanager Pod and StatefulSet Status

Verify that all Alertmanager instances are running and that there is no stalled rollout:

```bash
oc -n openshift-monitoring get pods -l app.kubernetes.io/name=alertmanager
oc -n openshift-monitoring get statefulset alertmanager-main
```

Check for pods in states such as:

- `CrashLoopBackOff`
- `Pending`
- `ContainerCreating`

A pod that has not successfully restarted may still be running with an older configuration.

### 3. Compare the Loaded Configuration

For each affected Alertmanager instance, compare the configuration currently loaded by Alertmanager:

```bash
oc -n openshift-monitoring exec alertmanager-main-0 -c alertmanager -- \
  curl -s http://localhost:9093/api/v2/status | jq -r '.config.original'
```

Repeat for the other Alertmanager instance:

```bash
oc -n openshift-monitoring exec alertmanager-main-1 -c alertmanager -- \
  curl -s http://localhost:9093/api/v2/status | jq -r '.config.original'
```

Compare the outputs to identify the configuration difference.

### 4. Check Config-Reloader Logs

If the configuration differs while all pods are `Running`, check the `config-reloader` container on the affected instance:

```bash
oc -n openshift-monitoring logs <affected-pod> -c config-reloader --since=1h
```

Look for configuration parsing, reload, or file-processing errors.

### 5. Check the Managed Configuration

If required, inspect the Alertmanager configuration Secret:

```bash
oc -n openshift-monitoring get secret alertmanager-main \
  -o jsonpath='{.data.alertmanager\.yaml}' | base64 -d
```

Compare the expected configuration with the configuration reported by the affected Alertmanager instance.

## Mitigation

### 1. Correct the Configuration

Identify which Alertmanager instance has the incorrect configuration and restore the expected configuration.

If the configuration was manually modified, revert the change and allow the Cluster Monitoring Operator and `config-reloader` to reconcile the configuration.

### 2. Resolve Pod or Config-Reloader Problems

If a pod is unable to load the current configuration, resolve the underlying issue reported by the pod or `config-reloader`.

For example: Pod may identify scheduling, volume-mount, image-pull, or container-startup issues.

```bash
oc -n openshift-monitoring describe pod <affected-pod>
oc -n openshift-monitoring logs <affected-pod> -c config-reloader
```

### 3. Restart a Stuck Alertmanager Instance

If the configuration is correct but an instance has failed to reload it, restart the affected pod:

```bash
oc -n openshift-monitoring delete pod <affected-pod>
```

The StatefulSet recreates the pod and it should load the current configuration.

### 4. Verify Recovery

Confirm that all Alertmanager instances report the same configuration hash:

```promql
alertmanager_config_hash{job=~"alertmanager-main|alertmanager-user-workload"}
```

The instances belonging to the same service should have the same `config_hash`.

The alert should clear after the configured `20m` pending period is no longer satisfied.
