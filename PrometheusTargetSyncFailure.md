# PrometheusTargetSyncFailure

**PrometheusRule Source:** cluster-monitoring-operator · **Pending For:** 5m · **Severity:** Critical · **Runbook:** PrometheusTargetSyncFailure

## Meaning

This alert is triggered when a Prometheus instance running in the `openshift-monitoring` namespace has failed to synchronize one or more targets.

Prometheus dynamically discovers and synchronizes scrape targets based on monitoring resources such as `ServiceMonitor`, `PodMonitor`, and `Probe`.

A target synchronization failure occurs when Prometheus is unable to create or synchronize a discovered target because of an invalid target configuration.

Examples include:

* Invalid target address
* Invalid target relabeling configuration
* Invalid regular expression
* Invalid target configuration generated from a monitoring resource

The alert remains **Pending for 5 minutes** before transitioning to **Firing**.

## Impact

* Metrics from affected targets will not be collected.
* Existing metrics for affected targets may become stale.
* Alerts that depend on the affected metrics may not evaluate correctly.
* Monitoring and observability of the affected cluster components may be degraded.

## Diagnosis

### 1. Inspect Prometheus Logs

Check the logs of the Prometheus pods in the `openshift-monitoring` namespace.

```bash
oc -n openshift-monitoring logs -l 'app.kubernetes.io/name=prometheus' -c prometheus |
  grep -iE "Creating target failed|error"
```

If a specific Prometheus pod needs to be investigated:

```bash
oc -n openshift-monitoring logs <prometheus-pod> -c prometheus |
  grep -iE "Creating target failed|error"
```

If the Prometheus container has restarted, check the previous container logs:

```bash
oc -n openshift-monitoring logs <prometheus-pod> -c prometheus --previous |
  grep -iE "Creating target failed|error"
```

**Verification:**

Look for messages similar to:

```text
Creating target failed
```

For example:

```text
Creating target failed ... err="instance 0 in group endpoints/openshift-monitoring/<resource>: no address"
```

The error message identifies the reason Prometheus failed to create or synchronize the target.

---

### 2. Confirm the Target Synchronization Failure

Verify that the target synchronization failure counter has increased:

```bash
oc exec -n openshift-monitoring prometheus-k8s-0 -- \
  curl -sG \
  --data-urlencode \
  'query=increase(prometheus_target_sync_failed_total{job="prometheus-k8s"}[30m]) > 0' \
  http://localhost:9090/api/v1/query |
  jq -r '.data.result[] |
    [.metric.pod,.metric.job,.metric.scrape_job,.value[1]] | @tsv'
```

**Verification:**

A result with a value greater than `0` confirms that Prometheus has recorded one or more target synchronization failures during the previous 30 minutes.

---

### 3. Identify the Faulty Resource

The Prometheus logs will typically identify the affected scrape pool or target.

Check the monitoring resources in `openshift-monitoring`:

```bash
oc get servicemonitor -n openshift-monitoring
```

```bash
oc get podmonitor -n openshift-monitoring
```

```bash
oc get probe -n openshift-monitoring
```

If the log identifies a specific resource, inspect it directly.

For a `ServiceMonitor`:

```bash
oc -n openshift-monitoring get servicemonitor <name> -o yaml
```

For a `PodMonitor`:

```bash
oc -n openshift-monitoring get podmonitor <name> -o yaml
```

For a `Probe`:

```bash
oc -n openshift-monitoring get probe <name> -o yaml
```

**Verification:**

Identify the monitoring resource corresponding to the scrape pool or target reported in the Prometheus logs and inspect its configuration.

## Mitigation

The mitigation depends on the error identified in the Prometheus logs.

| Scenario                                     | What to look for                                                                                        | Action / Mitigation                                                                                          |
| -------------------------------------------- | ------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------ |
| **Invalid target address**                   | Logs contain errors such as `no address` or an invalid target address                                   | Correct the affected `ServiceMonitor`, `PodMonitor`, or `Probe` so that a valid target address is generated. |
| **Invalid relabel configuration**            | Logs indicate an invalid regex, relabeling action, or target relabeling configuration                   | Correct the `relabelings` configuration in the affected monitoring resource.                                 |
| **Incorrect Service/Endpoint configuration** | The ServiceMonitor is selected but the associated target configuration is invalid                       | Verify the Service, Endpoints/EndpointSlice, port, selector, and ServiceMonitor configuration.               |
| **Invalid monitoring resource**              | The Prometheus logs identify a specific ServiceMonitor, PodMonitor, or Probe with a configuration error | Correct the identified resource according to the error reported in the logs.                                 |

### Example: Invalid Target Address

A target synchronization failure can occur when target relabeling produces an empty `__address__`.

Example Prometheus log:

```text
Creating target failed ... err="instance 0 in group endpoints/openshift-monitoring/target-sync-test: no address"
```

In this situation:

1. Identify the associated `ServiceMonitor`.
2. Inspect its `relabelings`.
3. Verify that `__address__` is not being replaced with an empty value.
4. Correct the configuration.
5. Verify that Prometheus successfully creates the target.

## Verification

### 1. Verify Prometheus Logs

After correcting the configuration:

```bash
oc -n openshift-monitoring logs <prometheus-pod> -c prometheus --since=10m |
  grep -iE "Creating target failed|error"
```

The target synchronization error should no longer be generated.

### 2. Verify Target Synchronization

Confirm that the target synchronization failure counter is no longer increasing:

```bash
oc exec -n openshift-monitoring prometheus-k8s-0 -- \
  curl -sG \
  --data-urlencode \
  'query=increase(prometheus_target_sync_failed_total{job="prometheus-k8s"}[30m])' \
  http://localhost:9090/api/v1/query |
  jq -r '.data.result[] |
    [.metric.pod,.metric.job,.metric.scrape_job,.value[1]] | @tsv'
```

### 3. Verify Alert Recovery

Check the alert state:

```bash
oc exec -n openshift-monitoring prometheus-k8s-0 -- \
  curl -sG \
  --data-urlencode \
  'query=ALERTS{alertname="PrometheusTargetSyncFailure"}' \
  http://localhost:9090/api/v1/query |
  jq -r '.data.result[] |
    [.metric.alertstate,.metric.pod,.value[1]] | @tsv'
```

The alert should eventually resolve once the target synchronization failure is no longer within the alert's evaluation window.

> **Important:** The alert expression uses a **30-minute lookback window**. Therefore, correcting or deleting the faulty resource does not necessarily clear the alert immediately. A previous target synchronization failure can continue to satisfy the alert expression until it falls outside the 30-minute window.

