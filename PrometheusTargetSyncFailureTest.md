# Reproduce `PrometheusTargetSyncFailure` Alert 

Reproduce the `PrometheusTargetSyncFailure` alert in the `openshift-monitoring` namespace by creating a ServiceMonitor whose relabeling configuration changes the target `__address__` to an empty value.

The test creates:

* A temporary Service
* A manually defined Endpoints object
* A ServiceMonitor
* A target relabeling rule that changes `__address__` to an empty value

---

## 1. Create the Test Resources

Run the following command:

```bash
cat <<'EOF' | oc apply -f -
apiVersion: v1
kind: Service
metadata:
  name: target-sync-test
  namespace: openshift-monitoring
  labels:
    app: target-sync-test
spec:
  ports:
  - name: metrics
    port: 8080
    targetPort: 8080
---
apiVersion: v1
kind: Endpoints
metadata:
  name: target-sync-test
  namespace: openshift-monitoring
subsets:
- addresses:
  - ip: 10.0.0.1
  ports:
  - name: metrics
    port: 8080
---
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: target-sync-test
  namespace: openshift-monitoring
spec:
  selector:
    matchLabels:
      app: target-sync-test
  endpoints:
  - port: metrics
    interval: 15s
    relabelings:
    - sourceLabels:
      - __address__
      regex: .+
      targetLabel: __address__
      replacement: ""
      action: replace
EOF
```


## 2. Verify the Test Resources

Run:

```bash
oc get svc target-sync-test -n openshift-monitoring

oc get endpoints target-sync-test -n openshift-monitoring

oc get servicemonitor target-sync-test -n openshift-monitoring
```

The Service, Endpoints, and ServiceMonitor should be present in `openshift-monitoring`.

---

## 3. Wait for 1-2 mins to Verify Prometheus Picked Up the ServiceMonitor


## 4. Verify the Prometheus Target Synchronization Error

Run:

```bash
oc -n openshift-monitoring logs -l 'app.kubernetes.io/name=prometheus' -c prometheus |
  grep -iE "Creating target failed|error"
```

### Expected result

Prometheus should report an error similar to:

```text
time=2026-09-15T11:34:22.916Z level=ERROR source=scrape.go:478 msg="Creating target failed" component="scrape manager" scrape_pool=serviceMonitor/openshift-monitoring/target-sync-test/0 err="instance 0 in group endpoints/openshift-monitoring/target-sync-test: no address"
time=2026-09-15T11:34:27.915Z level=ERROR source=scrape.go:478 msg="Creating target failed" component="scrape manager" scrape_pool=serviceMonitor/openshift-monitoring/target-sync-test/0 err="instance 0 in group endpoints/openshift-monitoring/target-sync-test: no address"

```

## 5. Verify the Alert

check by login to OpenShift console by navigating  Observe -> Alerting -> Pending|Firing

```text
PrometheusTargetSyncFailure
```

The alert should identify the affected Prometheus instance through its labels, including:

```text
namespace=openshift-monitoring
```

---

## 6. Clean Up the Test Resources

After completing the test, remove all resources created by this procedure:

```bash
oc delete servicemonitor target-sync-test -n openshift-monitoring

oc delete endpoints target-sync-test -n openshift-monitoring

oc delete service target-sync-te
```
