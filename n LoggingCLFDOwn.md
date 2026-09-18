# LoggingCLFDown

**PrometheusRule Source:** `cluster-logging-forwarder-endpoint-alerts`
**Alert Severity:** `Critical`
**Pending Period:** `5m`

## Meaning

This alert is triggered when Prometheus fails to scrape the `clf-otlp` target.

The Cluster Log Forwarder (CLF) runs as a Vector-based pipeline in the `openshift-logging` namespace. Prometheus scrapes metrics from the forwarder to monitor service availability.

The metric `up{job="clf-otlp"}` is used to track whether the forwarder scrape target is healthy.

If this alert fires, it indicates that **Prometheus failed to scrape the ****`clf-otlp`**** target**. This can occur when the CLF application is not running, the target has no endpoints, or Prometheus cannot reach the metrics endpoint.


### Expression

```bash
up{job="clf-otlp"} == 0
```

## Impact

When the `clf-otlp` forwarder is unreachable:

* Log forwarding to external destinations, specifically the OTLP and Splunk HEC outputs configured on the `clf-otlp` forwarder, may experience problems.
* Cluster and application observability is severely degraded because the pipeline is down.

## Diagnosis

Set the namespace:

```bash
export NAMESPACE="openshift-logging"
```

### Check Argo CD status first

Check Argo CD or your GitOps tool to find out why the application is not running or is not being re-deployed.

### Confirm Vector pods are running

```bash
oc -n $NAMESPACE get pods -l app.kubernetes.io/name=vector
```

### Check Vector pod logs

```bash
oc -n $NAMESPACE logs -l app.kubernetes.io/name=vector
```

Look for authentication, TLS, configuration, or connection errors.

## Mitigation

The mitigation action depends on the failure condition identified during diagnosis.

| Scenario                                                             | What to look for                                                                                      | Action / Mitigation                                                                                                                                                                                                      |
| -------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **Application removed or not re-deploying**                          | Check if the Argo CD application has been removed or deleted.                                         | Check Argo CD to find out why the application is not running. Investigate the deployment/Argo CD status before debugging endpoints.                                                                                      |
| **No Vector pod is Running and Ready or in in CrashLoopBackOff **    | Run `oc -n $NAMESPACE get pods`. Check whether pods are `Pending`, `CrashLoopBackOff`, or missing.    | Confirm the `clf-otlp` Cluster Log Forwarder pods are running in `openshift-logging` and check pod logs : Run `oc -n $NAMESPACE logs <vector-pod>` and resolve any identified authentication, TLS, or connection errors. |                                                                 |                                                                                                       |                                                                                                                                                                                                                          |
| **Vector pods are Running, but alert is firing (****`up == 0`****)** | Run `oc get endpoints clf-otlp -n openshift-logging` and `oc get networkpolicy -n openshift-logging`. | If endpoints exist, verify that Prometheus can reach the metrics port. Remove or modify any restrictive NetworkPolicy blocking the scrape.                                                                               |

## Verification

Verify that the Vector pods are running:

```bash
oc -n $NAMESPACE get pods -l app.kubernetes.io/name=vector
```

Verify that the `clf-otlp` service has endpoints:

```bash
oc -n $NAMESPACE get endpoints clf-otlp
```

Verify the Prometheus metric on OpenShift Console  -> Observe -> Metrics

```promql
up{job="clf-otlp"}
```

The expected value is:

```text
up{job="clf-otlp"} 1
```

Once Prometheus successfully scrapes the target, the alert should resolve automatically.
