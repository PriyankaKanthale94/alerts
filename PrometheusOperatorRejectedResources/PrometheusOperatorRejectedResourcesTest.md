## Reproduce PrometheusOperatorRejectedResources Alert

This procedure reproduces the `PrometheusOperatorRejectedResources` alert by introducing a `ServiceMonitor` containing an invalid regular expression (`*invalid-regex`).

The Prometheus Operator validates the custom resource using Go's `regexp` engine during its reconciliation loop. Because `*` without a preceding expression is invalid syntax, the Operator rejects the resource and exposes the rejection via the metric `prometheus_operator_managed_resources{state="rejected"}`.

---

## Procedure:

### 1. Create the Test Namespace

Create a dedicated test namespace for user workload monitoring:

```bash
oc create namespace test-rejected-resources
```

### 2. Deploy Malformed ServiceMonitors

Apply malformed `ServiceMonitor` resources to trigger rejection on both platform monitoring (`openshift-monitoring`) and user workload monitoring (`openshift-user-workload-monitoring`).

### For Platform Monitoring (`prometheus-operator` in `openshift-monitoring`)

Create a malformed `ServiceMonitor` in the `default` namespace:

```yaml
cat <<EOF | oc apply -f -
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: bad-servicemonitor-platform
  namespace: default
spec:
  selector:
    matchLabels:
      app: dummy-platform
  endpoints:
  - port: web
    relabelings:
    - action: keep
      regex: "*invalid-regex"
      sourceLabels: [__meta_kubernetes_pod_name]
EOF
```

### For User Workload Monitoring (`prometheus-operator` in `openshift-user-workload-monitoring`)

Create a malformed `ServiceMonitor` in your user workload namespace:

```yaml
cat <<EOF | oc apply -f -
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: bad-servicemonitor-user
  namespace: test-rejected-resources
spec:
  selector:
    matchLabels:
      app: dummy-user
  endpoints:
  - port: web
    relabelings:
    - action: keep
      regex: "*invalid-regex"
      sourceLabels: [__meta_kubernetes_pod_name]
EOF
```

### 3. Verify Operator Log Errors

Verify that the Prometheus Operator log records the rejection:

```bash
# Platform operator logs
oc logs -n openshift-monitoring -l app.kubernetes.io/name=prometheus-operator -c prometheus-operator | grep -E "error parsing regexp|rejected"

# User workload operator logs
oc logs -n openshift-user-workload-monitoring -l app.kubernetes.io/name=prometheus-operator -c prometheus-operator | grep -E "error parsing regexp|rejected"
```

*Expected log output:*
```text
error parsing regexp: missing argument to repetition operator: `*`
```

### 4. Monitor the Rejection Metrics

### Using OpenShift Web Console

1. Navigate to **Observe** → **Metrics**.
2. Run the alert expression:
   ```promql
   min_over_time(prometheus_operator_managed_resources{job="prometheus-operator",namespace=~"openshift-monitoring|openshift-user-workload-monitoring",state="rejected"}[5m]) > 0
   ```
3. Once the condition persists for **5 minutes**, verify the alert is firing under **Observe** → **Alerting** → search for `PrometheusOperatorRejectedResources`.

---

## Cleanup

### 1. Delete Malformed Resources

Delete the malformed `ServiceMonitor` objects:

```bash
oc delete servicemonitor bad-servicemonitor-platform -n default
oc delete servicemonitor bad-servicemonitor-user -n test-rejected-resources
```

### 2. Delete the Demo Namespace

```bash
oc delete namespace test-rejected-resources
```

### 3. Confirm Resolution

Verify that rejected resources return to `0`:

```bash
oc -n openshift-monitoring exec prometheus-k8s-0 -c prometheus -- \
curl -sg http://localhost:9090/api/v1/query \
--data-urlencode \
query="prometheus_operator_managed_resources{job=\"prometheus-operator\",state=\"rejected\"}" \
| jq -r ".data.result[] | [.metric.namespace, .metric.resource, .value[1]] | @tsv"
```

*Verification:* The metric values will return `0`, and the alert will automatically resolve after the 5-minute evaluation window clears.
