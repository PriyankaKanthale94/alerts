# Reproduce LoggingCLFDown Alert

This procedure reproduces the `LoggingCLFDown` alert by intentionally causing Prometheus scrape requests to the Cluster Log Forwarder (`clf-otlp`) to fail.

The test applies a Kubernetes `NetworkPolicy` that blocks all ingress traffic to the Vector pods. When Prometheus attempts to scrape the `clf-otlp` metrics endpoint, the request is blocked, causing the scrape to fail and the `up{job="clf-otlp"}` metric to become `0`.

## Requirements

Before starting this reproducer, ensure the following requirements are met:

1. **OpenShift Logging Operator** is installed and running.
2. A **ClusterLogForwarder** named `clf-otlp` is actively deployed in the `openshift-logging` namespace.
3. The custom `PrometheusRule` containing the `LoggingCLFDown` alert (`up{job="clf-otlp"} == 0` for `5m`) has been successfully applied to the cluster.
4. You have `cluster-admin` privileges to create `NetworkPolicy` resources and query metrics.

---

# Procedure

## 1. Verify Baseline Pipeline Health

Ensure the Vector pods are currently healthy and running:

```bash
oc get pods -n openshift-logging -l app.kubernetes.io/name=vector
```

Verify that Prometheus is successfully scraping the target. The expected value is `1`:

```bash
oc -n openshift-monitoring exec prometheus-k8s-0 -- \
curl -sg http://localhost:9090/api/v1/query \
--data-urlencode 'query=up{job="clf-otlp"}' \
| jq -r '.data.result[] | [.metric.__name__, .value[1]] | @tsv'
```

## 2. Create the Blocking NetworkPolicy

Create a `NetworkPolicy` that blocks all ingress traffic to the Vector pods.

Create a file named `deny-prometheus-scrape.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-prometheus-scrape
  namespace: openshift-logging
spec:
  podSelector:
    matchLabels:
      app.kubernetes.io/name: vector
  policyTypes:
    - Ingress
```

Apply the policy:

```bash
oc apply -f deny-prometheus-scrape.yaml
```

This policy blocks all ingress traffic to the selected Vector pods, including traffic from the Prometheus scraper.

## 3. Verify the Metric Drops to 0

Prometheus scrapes the target periodically. Wait approximately 30 to 60 seconds, then run the following query to confirm that the `up` metric has dropped to `0`:

```bash
oc -n openshift-monitoring exec prometheus-k8s-0 -- \
curl -sg http://localhost:9090/api/v1/query \
--data-urlencode 'query=up{job="clf-otlp"}' \
| jq -r '.data.result[] | [.metric.__name__, .value[1]] | @tsv'
```

Expected result:

```text
The value for up{job="clf-otlp"} should be 0.
```

## 4. Monitor the Alert Status


### Using the OpenShift Console

Navigate to:

**Observe → Alerting → Alerts**  `LoggingCLFDown`

After Prometheus observes the failed scrape, the alert should enter `Pending` and transition to `Firing` after the configured `5m` pending period.

# Cleanup

## 1. Delete the NetworkPolicy

Restore Prometheus's ability to scrape the metrics endpoint by deleting the `NetworkPolicy`:

```bash
oc delete networkpolicy deny-prometheus-scrape -n openshift-logging
```

## 2. Verify Restoration

Confirm that the `up` metric returns to `1` after the next successful scrape:

```bash
oc -n openshift-monitoring exec prometheus-k8s-0 -- \
curl -sg http://localhost:9090/api/v1/query \
--data-urlencode 'query=up{job="clf-otlp"}' \
| jq -r '.data.result[] | [.metric.__name__, .value[1]] | @tsv'
```

## 3 Check the alert: 

Check the OpenShift console to confirm that the `LoggingCLFDown` alert has automatically resolved.

The alert should transition from **Firing** to **Inactive** once Prometheus successfully scrapes the `clf-otlp` target and the alert expression is no longer true.
