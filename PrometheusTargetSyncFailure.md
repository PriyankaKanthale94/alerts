# PrometheusTargetSyncFailure

**PrometheusRule Source:** cluster-monitoring-operator · **Pending For:** 5m · **Severity:** Critical · [**Runbook:**] (https://github.com/openshift/runbooks/blob/master/alerts/cluster-monitoring-operator/PrometheusTargetSyncFailure.md) 

## Meaning

This alert is triggered when a Prometheus instance running in the `openshift-monitoring` namespace has failed to synchronize one or more targets.

Prometheus dynamically discovers and synchronizes scrape targets based on monitoring resources such as `ServiceMonitor`, `PodMonitor`, `Probe` or other configuration resource.

A `PrometheusTargetSyncFailure` occurs when Prometheus is unable to create or synchronize a discovered target because of an invalid target configuration.

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

### Variables (from alert)
```bash

NAMESPACE = labels.namespace
POD = labels.pod
```
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

If the Prometheus container has restarted, check the previous container logs using --previous/-p


**Verification:**

For example :

```text
Creating target failed ... err="instance 0 in group endpoints/<namespace>/<resource>: no address"
```

The error message identifies the reason Prometheus failed to create or synchronize the target.

---


### 2. Identify the Faulty Resource

Identify the affected monitoring resource from the scrape_pool value.

The resource may be located in openshift-monitoring or another namespace selected by prometheus-k8s.

Inspect and correct the corresponding ServiceMonitor, PodMonitor, Probe, or other configuration resource.

```bash
oc get servicemonitor|podmonitor|probe -n <namespace>
```

If the log identifies a specific resource, inspect it directly.

```bash
oc -n <namespace> get servicemonitor/podmonitor/probe <name> -o yaml
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
| **Unknown**                   | | Collect inspect file : oc adm inspect ns/openshift-monitoring 

## Verification

### 1. Verify Prometheus Logs

After correcting the configuration:

```bash
oc -n openshift-monitoring logs -l 'app.kubernetes.io/name=prometheus' -c prometheus |
  grep -iE "Creating target failed|error"
```

The target synchronization error should no longer be generated.


### 2. Verify Alert Recovery

Check the alert state by logging to OpenShift Console :  Observe -> Alerting 

The alert should eventually resolve once the target synchronization failure is no longer within the alert's evaluation window.

> **Important:** The alert expression uses a **30-minute lookback window**. Therefore, correcting or deleting the faulty resource does not necessarily clear the alert immediately. A previous target synchronization failure can continue to satisfy the alert expression until it falls outside the 30-minute window.

