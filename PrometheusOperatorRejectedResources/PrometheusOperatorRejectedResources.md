## PrometheusOperatorRejectedResources

The `PrometheusOperatorRejectedResources` alert fires when the Prometheus Operator detects and rejects invalid monitoring custom resource definitions (CRDs), including:
- `ServiceMonitor`
- `PodMonitor`
- `PrometheusRule`
- `AlertmanagerConfig`

The Prometheus Operator performs schema and semantic validation during its reconciliation loop. When an object fails validation, it marks that resource as rejected rather than propagating a broken configuration to Prometheus, Alertmanager, or Thanos Ruler.


 ### Expression:
  ```promql
  min_over_time(prometheus_operator_managed_resources{job="prometheus-operator",namespace=~"openshift-monitoring|openshift-user-workload-monitoring",state="rejected"}[5m]) > 0
  ```

---


### 2. Impact

- **Monitoring Blackout:** The rejected monitoring custom resources are skipped entirely. Their scrape targets or alerting/recording rules are omitted from the active configuration.
- **Silent Metric Loss:** Target workloads defined in rejected `ServiceMonitor` or `PodMonitor` objects will not be scraped.
- **Alert Inaction:** Rules inside rejected `PrometheusRule` definitions will not evaluate or trigger alerts.

---

### 3. Diagnosis

### Step 3.1: Identify Custom Resource Type and Scope

Identify whether the issue is with the platform stack or a user workload, and determine which custom resource type (`resource`) is failing.

#### Option A: Using the OpenShift Web Console

1. In the OpenShift Web Console, navigate to **Observe** → **Alerting**.
2. Search for the alert `PrometheusOperatorRejectedResources`.
3. Click the alert to inspect details and check the **Labels**:
   - `namespace`: Indicates either `openshift-monitoring` (platform) or `openshift-user-workload-monitoring` (user workloads).
   - `resource`: Indicates the resource kind (e.g., `servicemonitors`, `podmonitors`, `prometheusrules`, `alertmanagerconfigs`).

#### Option B: Using the CLI

Run the following command to retrieve active alerts directly from Alertmanager:

```bash
# Get Alertmanager route host
ALERTMANAGER=$(oc get route alertmanager-main -n openshift-monitoring -o jsonpath='{@.spec.host}')

# Fetch firing instances of PrometheusOperatorRejectedResources
curl -sk -H "Authorization: Bearer $(oc create token prometheus-k8s -n openshift-monitoring)" \
  "https://${ALERTMANAGER}/api/v2/alerts?filter=alertname=PrometheusOperatorRejectedResources" | jq .
```

---

### Step 3.2: Identify the Specific Rejected Resource and Cause

From OCP 4.16+, the Prometheus Operator records an `InvalidConfiguration` event on rejected resources.

- **Via Web Console:**
  1. Navigate to **Home** → **Events**.
  2. Set project filter to **All projects**.
  3. Filter by resource kind (e.g., `ServiceMonitor`, `PrometheusRule`).
  4. Look for warning events with reason `InvalidConfiguration`.

- **Via CLI:**
  Query cluster events for the specific resource kind identified in Step 3.1 (e.g., `ServiceMonitor`):

  ```bash
  oc get events --field-selector involvedObject.kind=ServiceMonitor -A --sort-by='.metadata.creationTimestamp'
  ```

  *Sample Output:*
  ```text
  NAMESPACE   LAST SEEN   TYPE      REASON                 OBJECT                       MESSAGE
  default     106s        Warning   InvalidConfiguration   servicemonitor/example-app   ServiceMonitor example-app was rejected due to invalid configuration: it accesses file system via bearer token file which Prometheus specification prohibits
  ```


### 4. Mitigation and Resolution

Mitigation depends on the rejected resource type and the error reported in the logs or events.

### 4.1. Issues with `ServiceMonitor` and `PodMonitor`

| Root Cause | Mitigation Steps |
| :--- | :--- |
| **Invalid Relabeling Configuration** (e.g., malformed regular expression) | Correct the regex syntax in `spec.endpoints[].relabelings` or `metricRelabelings`. Ensure Go `regexp` RE2 compatibility. |
| **Scrape Interval < Scrape Timeout** | Adjust `scrapeTimeout` to be less than or equal to `interval`. |
| **Invalid Secret or ConfigMap Reference** | Ensure referenced Secrets or ConfigMaps exist in the same namespace and the specified key exists. |
| **File System Access Rules Violation** (e.g., bearer token file or TLS cert path) | User-workload monitoring strictly forbids direct node filesystem paths (`bearerTokenFile`). Replace file paths with a Kubernetes Secret reference (`bearerTokenSecret`). |
| **Invalid TLS Configuration** | Verify `tlsConfig` parameters (e.g., `ca`, `cert`, `key`) point to valid in-namespace Secrets or ConfigMaps. |

#### Third-Party and Community Operators

If the rejected resource was created automatically by a third-party or community operator:

1. **Certified Red Hat Operator:**
   - If installed in `openshift-operators`, reinstall in an isolated operator namespace.
   - If installed in a dedicated namespace and documentation requires cluster monitoring, verify the namespace has the label `openshift.io/cluster-monitoring: "true"`.
   - Alternatively, exclude the resource from user-defined monitoring by labeling the namespace or the resource with:
     ```bash
     oc label <resource-kind> <resource-name> -n <namespace> openshift.io/user-monitoring="false"
     ```
2. **Community Operator:**
   - Exclude the resource or its namespace from monitoring using:
     ```bash
     oc label namespace <namespace> openshift.io/user-monitoring="false"
     ```

---

### 4.2. Issues with `AlertmanagerConfig`

- **Missing or Invalid Keys:** Check that referenced notification secret keys (e.g., Slack Webhook URLs, PagerDuty integration keys) exist.
- **Receiver/Route Errors:** Correct configuration syntax errors in `spec.receivers` or `spec.route`.
- **Unsupported Inhibition Rules:** Fix unsupported match expressions in inhibition rules.
- **Webhook Check:** The admission webhook should typically intercept invalid `AlertmanagerConfig` resources at submission time. If bad resources entered the cluster, verify the health of the webhook:
  ```bash
  oc get pods -n openshift-monitoring -l app.kubernetes.io/name=prometheus-operator-admission-webhook
  ```

---

### 4.3. Issues with `PrometheusRule`

- **PromQL Syntax Errors:** Fix invalid expressions in `expr`. Test the expression in the OpenShift Web Console (**Observe** → **Metrics**) before updating the rule.
- **Webhook Check:** As with `AlertmanagerConfig`, verify the admission webhook pods in `openshift-monitoring` are running and responsive.

---

### 5. Verification

After fixing or deleting the invalid custom resources:

1. Query Prometheus to ensure rejected resources counter returns to `0`:
   ```bash
   oc -n openshift-monitoring exec prometheus-k8s-0 -c prometheus -- \
     curl -sg http://localhost:9090/api/v1/query \
     --data-urlencode 'query=prometheus_operator_managed_resources{job="prometheus-operator",state="rejected"}' \
     | jq -r '.data.result[] | [.metric.namespace, .metric.controller, .metric.resource, .value[1]] | @tsv'
   ```
2. Confirm the alert resolves under **Observe** → **Alerting** once the 5-minute evaluation window passes.

---

### 6. References

- [Red Hat OpenShift Monitoring Documentation](https://docs.openshift.com/container-platform/latest/monitoring/understanding-the-monitoring-stack.html)
- [Red Hat Solution 6992399: PrometheusOperatorRejectedResources firing continuously](https://access.redhat.com/solutions/6992399)
