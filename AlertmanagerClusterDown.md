# AlertmanagerClusterDown

**PrometheusRule Source:** `openshift-monitoring` · **Pending For:** `5m` · **Severity:** `Warning` .

---

## Meaning

This alert indicates that half or more of the Alertmanager instances within a specific cluster (`alertmanager-main` or `alertmanager-user-workload`) have been unreachable or down for at least 5 minutes.

If Prometheus cannot scrape the `/metrics` endpoint of an Alertmanager pod, the `up` metric for that instance evaluates to `0`. When the average of the `up` metric falls below `0.5` over a 5-minute window for a given cluster, this alert is triggered.

Alert Expression : 
 `(count by (namespace, service) (avg_over_time(up{job=~"alertmanager-main\|alertmanager-user-workload"}[5m]) < 0.5) / count by (namespace, service) (up{job=~"alertmanager-main\|alertmanager-user-workload"})) >= 0.5` 

---

## Impact

* **Loss of High Availability:** If one of the two default instances is down, the system is running without redundancy.
* **Delayed or Dropped Alerts:** If all instances in the cluster go down, Prometheus will be unable to send firing alerts to Alertmanager, resulting in a total failure to route critical notifications to external receivers such as PagerDuty, Slack, or Email.
* **Silences Unavailable:** Users may not be able to create, view, or manage alert silences through the OpenShift UI.

---

## Diagnosis

The alert is typically caused by one of the following issues:

| Type               | Meaning                                                           | What to investigate                                                                     |
| ------------------ | ----------------------------------------------------------------- | --------------------------------------------------------------------------------------- |
| `Pod Failure`      | The Alertmanager pod is in a `CrashLoopBackOff` or `Error` state. | Container logs for configuration errors or OOM (Out of Memory) kills.                   |
| `Scheduling Issue` | The pod is stuck in a `Pending` state.                            | Node availability, resource constraints, or PVC (PersistentVolumeClaim) binding issues. |
| `Network Issue`    | The pod is `Running` but Prometheus cannot reach it.              | NetworkPolicies blocking traffic or CNI/SDN network disruptions.                        |

### 1. Identify the Failing Instances

Run the following PromQL query in the OpenShift Console under **Observe > Metrics** to determine which specific Alertmanager instances are down:

```promql
up{job=~"alertmanager-main|alertmanager-user-workload"} == 0
```

**Check the Alertmanager target**

In the OpenShift Console: Observe → Targets

Locate the affected Alertmanager target and review:
```
Target state
Last scrape
Scrape duration
Error message
```

### 2. Check the Status of Alertmanager Service , Endpoints and Pods. 

```bash
oc get svc alertmanager-main -n openshift-monitoring
```

```bash
oc get endpoints alertmanager-main -n openshift-monitoring
```

Also check EndpointSlices:

```bash
oc get endpointslice -n openshift-monitoring \
  -l kubernetes.io/service-name=alertmanager-main
```

Verify that the expected Alertmanager endpoints are present.

Check Alertmanager pods in the `openshift-monitoring` or `openshift-user-workload-monitoring` namespace.

```bash
oc get pods -n openshift-monitoring -l app.kubernetes.io/name=alertmanager
```

#### User-workload Alertmanager cluster

```bash
oc get pods -n openshift-user-workload-monitoring -l app.kubernetes.io/name=alertmanager
```

### 3. Inspect Pod Logs and Events

If a pod is in `CrashLoopBackOff` or failing to start, inspect its logs to identify configuration errors or crash reasons.

#### Check the logs for the specific failing pod

```bash
oc logs -n openshift-monitoring <alertmanager-pod-name> -c alertmanager
```

#### If the pod previously crashed, check the previous logs

```bash
oc logs -n openshift-monitoring <alertmanager-pod-name> -c alertmanager -p
```

Check the events associated with the pod for scheduling or readiness probe failures:

```bash
oc describe pod <alertmanager-pod-name> -n openshift-monitoring
```

---

## Mitigation

### 1. Resolve Configuration Errors

If the logs indicate a configuration error, such as a malformed `alertmanager.yaml` or invalid receiver settings, fix the configuration.

Check the Alertmanager configuration Secret in the `openshift-monitoring` namespace.

Revert any recent incorrect changes to the Alertmanager configuration.

### 2. Address Resource and Scheduling Issues

If the Alertmanager pod is `Pending`, investigate node resources, scheduling constraints, and storage.

#### Node resources

Ensure that sufficient CPU and memory are available on the worker nodes.

Check for:

* Cordoned nodes
* Node taints
* Resource pressure
* Insufficient CPU or memory
* Scheduling constraints

Useful commands 
```bash
oc get nodes
oc describe node
oc get pods -n openshift-monitoring -l app.kubernetes.io/name=alertmanager -o wide
oc adm top node | grep <node-name>
```

#### Storage

Check whether the associated PersistentVolumeClaims are bound and whether the underlying storage is functioning correctly.

```bash
oc get pvc -n openshift-monitoring | grep alertmanager
```

### 3. Resolve OOMKills

If the pod was killed due to an Out Of Memory condition:

1. Use `oc describe pod` and inspect the container state for `OOMKilled`.
2. Check the pod's resource requests and limits.
3. Review recent changes to monitoring configuration.
4. Adjust the Alertmanager resource configuration through the appropriate cluster monitoring configuration source if additional resources are required.

Example:

```bash
oc describe pod <alertmanager-pod-name> -n openshift-monitoring
```

Look for:

```text
Reason: OOMKilled
```

### 4. Network and Readiness Probes

If the Alertmanager pod is `Running` and `Ready` but the target
remains down, check whether a custom `NetworkPolicy` is blocking
Prometheus from reaching the Alertmanager metrics endpoint.

Check the policies in the namespace:

```bash
oc get networkpolicy -n openshift-monitoring
* Check for custom `NetworkPolicy` objects in the namespace that might unintentionally block ingress traffic from Prometheus.
* Verify that traffic to the Alertmanager metrics endpoint is allowed.
* Check the Alertmanager readiness probe.
* Review the pod events for readiness or liveness probe failures.
* Verify that Prometheus can reach the Alertmanager metrics endpoint.
```

Inspect the pod:

```bash
oc describe pod <alertmanager-pod-name> -n openshift-monitoring
```

Check NetworkPolicies:

```bash
oc get networkpolicy -n openshift-monitoring
```

If required, inspect a specific NetworkPolicy:

```bash
oc describe networkpolicy <networkpolicy-name> -n openshift-monitoring
```

---

## Recovery Verification

After resolving the underlying issue, verify that the affected Alertmanager instance is healthy and that Prometheus can scrape it successfully.

### Verify Alertmanager pods

```bash
oc get pods -n openshift-monitoring -l app.kubernetes.io/name=alertmanager
```

The expected state is:

```text
STATUS: Running
READY: 2/2
```

for each healthy Alertmanager pod.

### Verify the `up` metric

Run:

```promql
up{job=~"alertmanager-main|alertmanager-user-workload"}
```

Healthy Alertmanager instances should report:

```text
1
```

### Verify the alert

Confirm that `AlertmanagerClusterDown` is no longer firing in:

**Observe → Alerting**

The alert should transition from **Firing** to **Inactive** after the underlying condition has cleared and the configured alert evaluation period has elapsed.
