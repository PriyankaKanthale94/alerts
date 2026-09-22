# Reproduce AlertmanagerClusterDown Alert and PrometheusErrorSendingAlertsToSomeAlertmanagers

This procedure reproduces the `AlertmanagerClusterDown` for evaluating alert condition : ``Half or more of the Alertmanager instances within the same cluster are down`` by intentionally
causing Prometheus scrape requests to exactly half of the main Alertmanager instances to fail.

The test uses a `NetworkPolicy` applied to the `openshift-monitoring` namespace. 
By blocking ingress traffic specifically to the `alertmanager-main-0` pod, Prometheus 
is unable to scrape it. This causes the `up` metric for that specific pod to evaluate to `0`. 

This reproducer also generates PrometheusErrorSendingAlertsToSomeAlertmanagers Alerts as more than 1% of alerts sent by Prometheus to a specific Alertmanager are affected with error.

# Procedure:

## 1. Create the Targeted Network Policy

Apply the following `NetworkPolicy` to deny all ingress traffic specifically to the first pod of the main Alertmanager cluster:

```bash
cat <<EOF | oc apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: alertmanager-main-half-scrape-block
  namespace: openshift-monitoring
spec:
  podSelector:
    matchExpressions:
      - key: statefulset.kubernetes.io/pod-name
        operator: In
        values:
          - alertmanager-main-0
  policyTypes:
  - Ingress
  ingress: []
EOF
```

## 2. Verify the Network Policy

Confirm that the network policy was created successfully:

```bash
oc get networkpolicy alertmanager-main-half-scrape-block -n openshift-monitoring
```

## 3. Monitor the Status after 5 minutes:

Log into the OpenShift Console -> Observe -> Alerting -> AlertmanagerClusterDown

***Monitor the Alert Expression Evaluation (CLI)***

```bash
watch -n 30 '
oc -n openshift-monitoring exec prometheus-k8s-0 -- \
curl -sg http://localhost:9090/api/v1/query \
--data-urlencode \
query="(count by (namespace, service) (avg_over_time(up{job=~\"alertmanager-main|alertmanager-user-workload\"}[5m]) < 0.5) / count by (namespace, service) (up{job=~\"alertmanager-main|alertmanager-user-workload\"}))" \
| jq -r ".data.result[] | [.metric.service, .value[1]] | @tsv"
'
```

# Cleanup

## Delete the Network Policy

Remove the blocking policy to restore scrape traffic to the isolated pod:

```bash
oc delete networkpolicy alertmanager-main-half-scrape-block -n openshift-monitoring
```

## Verify Cleanup

After cleanup, verify that the network policy has been removed and the `up` metric returns to `1` for all instances:

```bash
oc get networkpolicy alertmanager-main-half-scrape-block -n openshift-monitoring
```

```bash
oc -n openshift-monitoring exec prometheus-k8s-0 -- \
curl -sg http://localhost:9090/api/v1/query \
--data-urlencode \
query="up{job=\"alertmanager-main\"}" \
| jq -r ".data.result[] | [.metric.pod, .value[1]] | @tsv"
```
