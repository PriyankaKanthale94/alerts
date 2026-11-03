# PrometheusDuplicateTimestamps

PrometheusRule Source: `openshift-monitoring/k8s` · Alert Severity: `Warning` · Pending Period : `1h` . [Runbook](https://github.com/openshift/runbooks/blob/master/alerts/cluster-monitoring-operator/PrometheusDuplicateTimestamps.md)

The `PrometheusDuplicateTimestamps` alert is triggered when there is a constant increase in dropped samples due to them having identical timestamps. 


Expression:

```bash
rate(prometheus_target_scrapes_sample_duplicate_timestamp_total{job=~"prometheus-k8s|prometheus-user-workload"}[5m]) > 0
```

---

## Impact:

When Prometheus drops duplicate timestamps:

- Unwanted samples might be scraped, while the desired samples are dropped and lost.
- Consequently, PromQL queries for the affected metrics may yield incorrect, incomplete, and unexpected results.
- Dashboards relying on the affected targets will display inaccurate data.

---

## Diagnosis:

1. Determine whether the alert has triggered for the instance of Prometheus used for default cluster monitoring or for the instance that monitors user-defined projects by viewing the alert message's `namespace` label:
   ```bash
   export NAMESPACE='<value of namespace label from alert>'
   ```
   *(Note: `openshift-monitoring` is for default cluster monitoring, and `openshift-user-workload-monitoring` is for user workloads).*

2. Review the standard warning logs for the affected Prometheus instance to identify the specific scrape pool and target:
   ```bash
   oc -n $NAMESPACE logs -l 'app.kubernetes.io/name=prometheus' | \
   grep 'Error on ingesting samples with different value but same timestamp.*' \
   | sort | uniq -c | sort -n
   ```
   *Look for the `scrape_pool` and `target` labels in the output.*


3. Enable debug log level on the Prometheus instance
As the example from the Diagnostic Steps is indicating that the Prometheus instance is that running in the openshift-namespace, edit the cluster-monitoring-config configmap to add logLevel: debug:

    ```bash
    $ oc edit cm cluster-monitoring-config -n openshift-monitoring 
    And add inside the component prometheusK8s (for the User Workload, used prometheus instead of prometheusK8s) the lines logLevel: debug as:

    ...
    data:
      config.yaml: |
        prometheusK8s:
          logLevel: debug 
    ...
    ```

5. Review the new debug logs to reveal the problematic metrics and series:
   ```bash
   oc -n $NAMESPACE logs -l 'app.kubernetes.io/name=prometheus' | \
   grep 'Duplicate sample for timestamp.*' | sort | uniq -c | sort -n
   ```
   *Look for the `series="<metric-name>"` label in the output.*

6. Disable the `debug` log level once the diagnosis is complete.

---

## Mitigation:

The mitigation action depends on the failure condition identified during diagnosis. Follow the applicable action plan below.

| Scenario | What to look for | Action / Mitigation |
|---|---|---|
| **Alert originates from `openshift-monitoring` namespace** | Check the alert's `namespace` label. | Check the Refer section or it might be a potential bug in default OpenShift cluster monitoring components contact support. |
| **Target duplication across different pools** | Review Prometheus warning logs. Look for the same target defined in different scrape pools, or distinct targets across pools producing the exact same samples. | Use the logs to locate where the conflicting targets are defined. Remove the duplicated targets or ensure that distinct targets are labeled uniquely (e.g., in their `ServiceMonitor` or `PodMonitor`). |
| **Target is exposing duplicated samples** | Review Prometheus debug logs. Look for the `Duplicate sample for timestamp` message and note the specific `series` being exposed multiple times. | Fix the metrics exposition in the broken target's application code. Ensure the application does not expose the exact same metric series multiple times in a single scrape response. |

---

## Refer
https://access.redhat.com/solutions/7101689 

https://access.redhat.com/solutions/7127615

https://access.redhat.com/solutions/7127014

https://access.redhat.com/solutions/7127972


```
