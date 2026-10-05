# AlertmanagerMembersInconsistent

**PrometheusRule Source:** `` · **Alert Severity:** `Warning` · **Pending Period:** `15m` · [Runbook](https://github.com/prometheus-operator/runbooks/blob/main/content/runbooks/alertmanager/AlertmanagerMembersInconsistent.md)

## Meaning

This alert indicates that one or more Alertmanager instances have not discovered all members of the Alertmanager cluster.

Alertmanager instances use the gossip protocol over TCP/UDP port `9094` to maintain cluster membership and synchronize state.

> **Note:** The alert expression applies to both `alertmanager-main` and `alertmanager-user-workload`. The checks below use `alertmanager-main` as an example. If the alert is firing for `alertmanager-user-workload`, perform the same checks against the `alertmanager-user-workload` Service, EndpointSlices, pods, NetworkPolicies, and logs.

## Impact

- Alert notifications may be duplicated.
- Silences and notification state may not be synchronized.
- Alertmanager high availability is degraded.

## Diagnosis

### 1. Check cluster membership

```bash
oc -n openshift-monitoring exec prometheus-k8s-0 -- \
curl -sg http://localhost:9090/api/v1/query \
--data-urlencode \
'query=alertmanager_cluster_members{job=~"alertmanager-main|alertmanager-user-workload"}' \
| jq -r '.data.result[] | [.metric.job, .metric.pod, .value[1]] | @tsv'
```

Compare the membership count reported by each Alertmanager instance.

### 2. Check Service endpoints and pod IPs

For `alertmanager-main`:

```bash
oc get endpoints alertmanager-main -n openshift-monitoring -o wide
```

```bash
oc get pods -n openshift-monitoring \
-l app.kubernetes.io/name=alertmanager,app.kubernetes.io/instance=main -o wide
```

For `alertmanager-user-workload`:

```bash
oc get endpoints alertmanager-user-workload -n openshift-user-workload-monitoring -o wide
```

```bash
oc get pods -n openshift-user-workload-monitoring \
-l app.kubernetes.io/name=alertmanager,app.kubernetes.io/instance=user-workload -o wide
```

Verify that the Service endpoint IPs match the current Alertmanager pod IPs.

### 3. Check EndpointSlices

For `alertmanager-main`:

```bash
oc get endpointslice -n openshift-monitoring \
-l kubernetes.io/service-name=alertmanager-main -o wide
```

For `alertmanager-user-workload`:

```bash
oc get endpointslice -n openshift-user-workload-monitoring \
-l kubernetes.io/service-name=alertmanager-user-workload -o wide
```

Verify that the expected Alertmanager pods and IP addresses are present.

### 4. Check NetworkPolicies

Check NetworkPolicies in the namespace containing the affected Alertmanager cluster:

```bash
oc get networkpolicy -n openshift-monitoring
```

or:

```bash
oc get networkpolicy -n openshift-user-workload-monitoring
```

Check whether any NetworkPolicy blocks Alertmanager gossip traffic over TCP/UDP `9094`.

### 5. Check pod readiness and restarts

Check the affected Alertmanager pods for readiness failures, restarts, or recent IP changes.

### 6. Check Alertmanager logs

```bash
oc logs <alertmanager-pod> -n <namespace> | \
grep -Ei 'gossip|cluster|memberlist'
```

Look for errors related to cluster communication, peer discovery, or membership.

### 7. Check node placement

```bash
oc get pods -n <namespace> -l app.kubernetes.io/name=alertmanager -o wide
```

If affected instances are on different nodes, check for node-level or network connectivity problems between those nodes.

## Mitigation

| Cause | Action |
|---|---|
| NetworkPolicy blocks TCP/UDP `9094` | Allow Alertmanager gossip traffic on TCP/UDP `9094`. |
| Network/CNI connectivity failure | Resolve the underlying network connectivity issue. |
| Incorrect Service Endpoint IP | Delete the incorrect Endpoint and allow it to be recreated with the correct IP. |
| Pod unavailable or not ready | Resolve the pod readiness or availability issue. |
| Stale Endpoint/EndpointSlice information | Correct the endpoint information and verify that it matches current pod IPs. |
| Alertmanager cluster communication errors | Review Alertmanager logs and resolve the reported cluster/memberlist issue. |

For an incorrect Endpoint:

```bash
oc delete endpoint <endpoint-name> -n <namespace>
```

Then verify:

```bash
oc get endpoints <endpoint-name> -n <namespace> -o wide
```

After remediation, verify that all Alertmanager instances report the expected cluster membership and that `AlertmanagerMembersInconsistent` clears.
