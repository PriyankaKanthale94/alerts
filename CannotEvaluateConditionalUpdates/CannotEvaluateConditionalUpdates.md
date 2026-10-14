# CannotEvaluateConditionalUpdates

**PrometheusRule Source:** `cluster-version-operator` · **Source:** `Platform` · **Pending For:** `-` · **Severity:** `Warning`

---

This alert indicates that the Cluster Version Operator (CVO) cannot evaluate conditional update risk rules and determine whether an update path is recommended for the cluster.

CVO evaluates PromQL expressions associated with conditional update risks using the monitoring query service. If evaluation fails, the corresponding `Recommended` condition becomes `Unknown`, with the reason `EvaluationFailed`.

The alert fires when this condition remains `Unknown` for at least one hour.

## Expression

```promql
max by (version, condition, status, reason) (
  (
    time() -
    cluster_version_conditional_update_condition_seconds{
      condition="Recommended",
      status="Unknown"
    }
  ) >= 3600
)
```

## Impact

- **Unknown update risk:** The cluster cannot reliably determine whether an update path is recommended.
- **Reduced upgrade visibility:** The CLI and console may report update recommendations as evaluation failures.
- **Upgrade automation:** Pipelines that require verified update recommendations may pause or fail.

## Diagnosis

### 1. Check the alert and its duration

```bash
oc -n openshift-monitoring exec prometheus-k8s-0 -c prometheus -- \
  promtool query instant http://localhost:9090 \
  'time() - cluster_version_conditional_update_condition_seconds{condition="Recommended",status="Unknown"}'
```

### 2. Inspect ClusterVersion conditions

```bash
oc get clusterversion version -o json | \
  jq '.status.conditionalUpdates[] |
      {version: .release.version, conditions: .conditions}'
```

Look for `type: Recommended`, `status: Unknown`, and `reason: EvaluationFailed`.

List the affected update risks and matching rules:

```bash
oc get clusterversion version -o json | \
  jq '.status.conditionalUpdates[] |
      {version: .release.version,
       risks: [.risks[] |
         {name: .name, message: .message, matchingRules: .matchingRules}]}'
```

### 3. Check CVO logs

```bash
oc logs -n openshift-cluster-version \
  deployment/cluster-version-operator \
  -c cluster-version-operator --since=1h |
  grep -iE 'conditional|evaluationfailed|promql|thanos|timeout|forbidden'
```

Look for errors such as:

- `connection refused` or service not found
- `context deadline exceeded`
- `403 Forbidden` or `Unauthorized`
- PromQL parse or query evaluation errors

### 4. Check monitoring components and endpoints

```bash
oc get pods -n openshift-monitoring \
  -l app.kubernetes.io/name=thanos-query

oc get pods -n openshift-monitoring \
  -l app.kubernetes.io/name=prometheus

oc get svc thanos-querier prometheus-k8s \
  -n openshift-monitoring

oc get endpointslice -n openshift-monitoring \
  -l kubernetes.io/service-name=thanos-querier

oc get endpointslice -n openshift-monitoring \
  -l kubernetes.io/service-name=prometheus-k8s
```

Check that the relevant pods are healthy and the Services have usable endpoints.

### 5. Check connectivity restrictions

```bash
oc get networkpolicy -n openshift-cluster-version
oc get networkpolicy -n openshift-monitoring
```

If logs indicate an authorization failure, investigate the applicable service account permissions and monitoring endpoint authentication. If a query or connection fails, inspect the associated service, endpoint, and network-policy configuration.

## Mitigation

| Scenario | Action / Mitigation | Verification |
|---|---|---|
| Monitoring pods are unhealthy | Check pod events and logs. Resolve the reported issue, such as a failed mount, scheduling error, or crash. | `oc get pods -n openshift-monitoring` |
| Thanos Querier has no endpoints | Verify that the Service selector matches the intended ready pods. Restore the original selector if it was modified. | `oc get endpointslice -n openshift-monitoring -l kubernetes.io/service-name=thanos-query` |
| NetworkPolicy blocks CVO queries | Update the blocking policy to allow the required CVO-to-monitoring traffic on the affected endpoint and port. | Confirm CVO logs no longer report connection timeouts. |
| Monitoring query returns 401/403 | Correct the confirmed authentication or authorization misconfiguration for the affected query endpoint. | Confirm CVO logs no longer report authorization errors. |
| Conditional update PromQL evaluation fails | If the risk expression is malformed or the update graph provides invalid rules, collect the affected `matchingRules` and CVO logs and contact Red Hat Support. | Confirm the condition no longer reports `EvaluationFailed`. |

### Upgrade workaround

If an upgrade is urgent, review the affected risk descriptions and advisory URLs in `.status.conditionalUpdates[].risks[]`. Assess the risk with the cluster and application owners before proceeding.
 
If proceeding with a non-recommended update is necessary, use the supported procedure for the target OpenShift release and record the accepted risk.

##  Verification

```bash
oc get clusterversion version -o json | \
  jq '.status.conditionalUpdates[] |
      {version: .release.version, conditions: .conditions}'
```

Verify that the affected `Recommended` condition is no longer `Unknown` because of `EvaluationFailed`.

Confirm the alert is no longer active.

