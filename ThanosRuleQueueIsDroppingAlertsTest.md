# Reproduce ThanosRuleQueueIsDroppingAlerts Alert

This procedure reproduces the `ThanosRuleQueueIsDroppingAlerts` alert by generating a large number of alert instances from a single PrometheusRule. The test environment creates a lightweight Python metric generator that exposes 15,000 distinct time series. The metrics are scraped by Prometheus and a PrometheusRule is used to generate 15,000 concurrent alert instances, causing the alert queue to become saturated and alerts to be dropped.

1. Create the Test Namespace:

Create a dedicated namespace for the test resources.

Bash

```bash
oc create namespace thanos-alert-test
```

2. Deploy the metric generator:

Deploy a lightweight Python web server that outputs 15,000 distinct time series (`dummy_metric{id="0"}` to `dummy_metric{id="14999"}`) whenever Prometheus scrapes it, along with the Service to route to it.

Bash

```bash
cat <<EOF | oc apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: massive-metric-generator
  namespace: thanos-alert-test
  labels:
    app: massive-metrics
spec:
  containers:
  - name: generator
    image: registry.access.redhat.com/ubi8/python-39:latest
    command: ["python"]
    args:
    - "-c"
    - |
      import http.server
      class Metrics(http.server.BaseHTTPRequestHandler):
          def do_GET(self):
              self.send_response(200)
              self.end_headers()
              self.wfile.write(b"".join(f"dummy_metric{{id=\"{i}\"}} 1\n".encode() for i in range(15000)))
      http.server.HTTPServer(("0.0.0.0", 8080), Metrics).serve_forever()
    ports:
    - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: massive-metrics-svc
  namespace: thanos-alert-test
  labels:
    app: massive-metrics
spec:
  ports:
  - port: 8080
    targetPort: 8080
    name: web
  selector:
    app: massive-metrics
EOF
```

3. Create the ServiceMonitor:

Instructs OpenShift to collect the metrics. The User Workload Monitoring stack scrape the Python pod every 10 seconds.

Bash

```bash
cat <<EOF | oc apply -f -
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: massive-metrics-monitor
  namespace: thanos-alert-test
spec:
  endpoints:
  - port: web
    interval: 10s
  selector:
    matchLabels:
      app: massive-metrics
EOF
```

4. Deploy a single, massive PrometheusRule: (Alert - DropTestAlert)

Bash

```bash
cat <<EOF | oc apply -f -
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: trigger-queue-drop
  namespace: thanos-alert-test
spec:
  groups:
  - name: heavy-alerts
    interval: 10s
    rules:
    - alert: DropTestAlert
      expr: dummy_metric > 0
EOF
```

Wait about 2 to 3 minutes for Prometheus to scrape the pod and Thanos Ruler to evaluate the new rule group.

5.  Review the logs for the Thanos Ruler pods:

```bash
oc -n openshift-user-workload-monitoring logs -l 'thanos-ruler=user-workload'
ts=2026-09-16T09:03:24.810322228Z caller=alert.go:177 level=warn msg="Alert batch larger than queue capacity, dropping alerts" numDropped=5000
ts=2026-09-16T09:04:34.843860786Z caller=alert.go:177 level=warn msg="Alert batch larger than queue capacity, dropping alerts" numDropped=5000
ts=2026-09-16T09:05:44.806526495Z caller=alert.go:177 level=warn msg="Alert batch larger than queue capacity, dropping alerts" numDropped=5000
```


6. Validate the ThanosRuleQueueIsDroppingAlerts alert:

Login to the OpenShift Console and navigate to: **Observe → Alerting**

Search for:

```text
ThanosRuleQueueIsDroppingAlerts
```

You will see number of DropTestAlert alerts as a part of this reproducer.
```text
DropTestAlertxxx
```
Because of this huge amount of alerts, the altering UI might be unresponsive. In that case check the alerts using  : 

```bash
oc exec -n openshift-monitoring alertmanager-main-0 -c alertmanager -- amtool alert query alertname="ThanosRuleQueueIsDroppingAlerts" --alertmanager.url="http://localhost:9093"
```

7. Clean up the test:

After validating the alert, delete the test namespace. 
Bash

```bash
oc delete namespace thanos-alert-test
```
