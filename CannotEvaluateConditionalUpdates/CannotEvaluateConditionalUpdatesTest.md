# Reproduce CannotEvaluateConditionalUpdates Alert

This procedure reproduces the `CannotEvaluateConditionalUpdates` alert by forcing the Cluster Version Operator (CVO) to fail its evaluation of conditional update risk rules.

The procedure switches the cluster update channel to one containing conditional upgrade edges (for example, `fast-4.22`) and scales down `thanos-querier` in the `openshift-monitoring` namespace. When CVO cannot query Prometheus metrics to evaluate PromQL risk conditions, it sets the conditional update condition to `Recommended: Unknown` with reason `EvaluationFailed`.

When this state persists continuously for 1 hour (3600 seconds), Prometheus fires the `CannotEvaluateConditionalUpdates` alert.

**Warning: Impact on the OpenShift Web Console**

Scaling down `thanos-querier` causes the OpenShift Web Console (**Observe → Alerting, Metrics, and Dashboards**) to fail with `502 Bad Gateway` or the message `You don't have access to this section due to cluster policy`.

This occurs because the Web Console proxies telemetry requests through `thanos-querier`. Prometheus and Alertmanager continue operating in the background. Monitor the alert using the CLI methods.

## Procedure

### 1. Switch to a Channel with Conditional Updates

CVO attempts risk evaluation when conditional update edges exist in the update graph. Switch to a channel that exposes conditional updates, such as `fast-4.22`.

```bash
oc adm upgrade channel fast-4.22
```

Confirm that conditional updates are present:

```bash
oc adm upgrade --include-not-recommended
```

Expected output includes releases listed under **Updates with known issues**:

```text
Updates with known issues:
  Version: 4.22.17
  Reason: MultiArchPayloadMissingOpenVSwitch
  ...
```

### 2. Unmanage the Cluster Monitoring Operator

CMO automatically reconciles and scales monitoring components back up unless an override is configured. Mark the Cluster Monitoring Operator deployment as unmanaged:

```bash
oc patch clusterversion version --type=merge \
  -p '{"spec":{"overrides":[{"group":"apps","kind":"Deployment","name":"cluster-monitoring-operator","namespace":"openshift-monitoring","unmanaged":true}]}}'
```

### 3. Scale Down CMO and Thanos Querier

```bash
oc scale deployment/cluster-monitoring-operator \
  -n openshift-monitoring --replicas=0

oc scale deployment/thanos-querier \
  -n openshift-monitoring --replicas=0
```

### 4. Restart CVO

CVO caches successful evaluation results in memory. Restart the CVO pod to clear the cache and force re-evaluation:

```bash
oc delete pod -l k8s-app=cluster-version-operator \
  -n openshift-cluster-version
```

Wait approximately 45 seconds for CVO to initialize and attempt the failed PromQL queries.

### 5. Verify the Evaluation Failure State

Confirm that the update status has changed to `Recommended: Unknown` with reason `EvaluationFailed`:

```bash
oc adm upgrade --include-not-recommended
```

Inspect the `status.conditionalUpdates` field:

```bash
oc get clusterversion version -o json |
  jq '.status.conditionalUpdates[] |
      {version: .release.version, conditions: .conditions}'
```

Expected output:

```json
{
  "conditions": [
    {
      "message": "Exposure to MultiArchPayloadMissingOpenVSwitch is unknown due to an evaluation failure...",
      "reason": "EvaluationFailed",
      "status": "Unknown",
      "type": "Recommended"
    }
  ],
  "version": "4.22.17"
}
```

### 6. Monitor Alert Status Using the CLI

Because the Web Console may return a `502 Bad Gateway` while `thanos-querier` is down, use the following commands to monitor Prometheus and Alertmanager directly.

#### A. Monitor Elapsed Failure Time

Track the elapsed seconds toward the 3600-second alert threshold:

```bash
watch -n 30 '
oc -n openshift-monitoring exec -c prometheus prometheus-k8s-0 -- \
  promtool query instant http://localhost:9090 \
  "time() - cluster_version_conditional_update_condition_seconds{condition=\"Recommended\",status=\"Unknown\"}"
'
```

#### B. Validate the Alert Expression Without Waiting One Hour

Use a reduced threshold, such as 10 seconds, to verify that the expression evaluates to true:

```bash
oc -n openshift-monitoring exec -c prometheus prometheus-k8s-0 -- \
  promtool query instant http://localhost:9090 \
  'max by (version, condition, status, reason) ((time() - cluster_version_conditional_update_condition_seconds{condition="Recommended",status="Unknown"}) >= 10)'
```

Expected output:

```text
{condition="Recommended", reason="EvaluationFailed", status="Unknown", version="4.22.16"} => 1
{condition="Recommended", reason="EvaluationFailed", status="Unknown", version="4.22.17"} => 1
```

#### C. Check the Alert State in Prometheus

Check whether the alert is pending or firing:

```bash
watch -n 30 '
oc -n openshift-monitoring exec -c prometheus prometheus-k8s-0 -- \
  promtool query instant http://localhost:9090 \
  "ALERTS{alertname=\"CannotEvaluateConditionalUpdates\"}"
'
```

- `alertstate="pending"`: The alert condition is active, and Prometheus is counting toward the configured one-hour `for` duration.
- `alertstate="firing"`: The alert has satisfied the one-hour pending duration and is firing.

#### D. Check Alertmanager

Query Alertmanager to verify that the alert has reached it:

```bash
oc -n openshift-monitoring exec -c alertmanager alertmanager-main-0 -- \
  amtool alert --alertmanager.url=http://localhost:9093 \
  query alertname=CannotEvaluateConditionalUpdates
```

## Cleanup

### 1. Remove the ClusterVersion Override

Remove the override so CVO resumes managing the monitoring operator:

```bash
oc patch clusterversion version --type=json \
  -p '[{"op":"remove","path":"/spec/overrides"}]'
```

**Note:** This removes the entire `spec.overrides` field. If other overrides exist, remove only the CMO override instead.

### 2. Restore CMO and Thanos Querier

Restore the deployments to their expected replica counts:

```bash
oc scale deployment/cluster-monitoring-operator \
  -n openshift-monitoring --replicas=1

oc scale deployment/thanos-querier \
  -n openshift-monitoring --replicas=2
```

### 3. Restore the Original Update Channel

Return the cluster to its baseline channel:

```bash
oc adm upgrade channel stable-4.21
```

### 4. Restart CVO

Restart CVO to trigger a fresh evaluation against the restored query service:

```bash
oc delete pod -l k8s-app=cluster-version-operator \
  -n openshift-cluster-version
```

### 5. Verify Recovery

Confirm that `thanos-querier` pods are running:

```bash
oc get pods -n openshift-monitoring 
```

Verify that the alert has cleared from Prometheus:

```bash
oc -n openshift-monitoring exec -c prometheus prometheus-k8s-0 -- \
  promtool query instant http://localhost:9090 \
  'ALERTS{alertname="CannotEvaluateConditionalUpdates"}'
```

Confirm the cluster update status:

```bash
oc adm upgrade
```

Finally, access the OpenShift Web Console and verify that **Observe → Alerting, Metrics, and Dashboards** is working normally again.
