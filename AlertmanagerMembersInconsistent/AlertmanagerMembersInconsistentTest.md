# Reproduce AlertmanagerMembersInconsistent Alert

This procedure reproduces the `AlertmanagerMembersInconsistent` alert by isolating one Alertmanager instance from the other cluster members.

## 1. Create the NetworkPolicy

Create a file named `block-am-gossip.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: block-am-gossip
  namespace: openshift-monitoring
spec:
  podSelector:
    matchLabels:
      statefulset.kubernetes.io/pod-name: alertmanager-main-0
  policyTypes:
  - Egress
  egress:
  - ports:
    - protocol: TCP
      port: 1
      endPort: 9093
    - protocol: UDP
      port: 1
      endPort: 9093
    - protocol: TCP
      port: 9095
      endPort: 65535
    - protocol: UDP
      port: 9095
      endPort: 65535
```

## 2. Apply the NetworkPolicy

```bash
oc apply -f block-am-gossip.yaml
```

## 3. Verify Cluster Membership

Check the membership reported by each Alertmanager:

```bash
oc -n openshift-monitoring exec prometheus-k8s-0 -- \
curl -sg http://localhost:9090/api/v1/query \
--data-urlencode \
'query=alertmanager_cluster_members{job="alertmanager-main"}' \
| jq -r '.data.result[] | [.metric.pod, .value[1]] | @tsv'
```

The isolated `alertmanager-main-0` should report fewer members than the other Alertmanager instance.

For example:

```text
alertmanager-main-0    1
alertmanager-main-1    2
```

## 4. Verify Alertmanager Pods Remain Up

```bash
oc -n openshift-monitoring exec prometheus-k8s-0 -- \
curl -sg http://localhost:9090/api/v1/query \
--data-urlencode \
'query=up{job="alertmanager-main"}' \
| jq -r '.data.result[] | [.metric.pod, .value[1]] | @tsv'
```

Both Alertmanager instances should report `1`.

## 5. Verify the Alert

The alert has a `15m` pending period. Keep the NetworkPolicy applied until the alert fires.

```bash
oc -n openshift-monitoring exec prometheus-k8s-0 -- \
curl -sg http://localhost:9090/api/v1/alerts \
| jq '.data.alerts[] | select(.labels.alertname=="AlertmanagerMembersInconsistent")'
```

The `AlertmanagerMembersInconsistent` alert should transition to `firing`.

## 6. Cleanup

Remove the NetworkPolicy:

```bash
oc delete networkpolicy block-am-gossip \
  -n openshift-monitoring
```
