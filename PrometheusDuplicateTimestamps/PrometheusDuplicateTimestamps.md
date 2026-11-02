## Reproduce PrometheusDuplicateTimestamps Alert

This procedure reproduces the `PrometheusDuplicateTimestamps` alert by intentionally
creating a target that violates the Prometheus exposition format. It exposes the
exact same metric series (identical metric name and labels) multiple times in a
single scrape response.

When Prometheus scrapes this endpoint, it assigns the exact same scrape timestamp
to both lines, causing ingestion to fail for the duplicate and incrementing the
`prometheus_target_scrapes_sample_duplicate_timestamp_total` metric.

## Requirements

 Prometheus Operator installed and User Workload Monitoring must be enabled

## Procedure:

### 1. Create a Custom Namespace

Create a dedicated namespace for this reproducer:

```bash
oc create namespace prometheus-demo-ns
```

### 2. Create the Faulty Application Script

Create a `ConfigMap` containing a simple Python HTTP server that serves duplicated metrics:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: duplicate-metrics-script
  namespace: prometheus-demo-ns
  labels:
    app: duplicate-metrics
data:
  server.py: |
    from http.server import BaseHTTPRequestHandler, HTTPServer
    
    class MetricsHandler(BaseHTTPRequestHandler):
        def do_GET(self):
            self.send_response(200)
            self.send_header("Content-type", "text/plain")
            self.end_headers()
            # Emitting the exact same series twice with different values
            self.wfile.write(b"my_custom_metric{label=\"test\"} 1.0\n")
            self.wfile.write(b"my_custom_metric{label=\"test\"} 2.0\n")
            
    if __name__ == "__main__":
        print("Starting faulty metrics server on port 8080...")
        server = HTTPServer(("0.0.0.0", 8080), MetricsHandler)
        server.serve_forever()
```

Apply the ConfigMap:

```bash
oc apply -f duplicate-metrics-configmap.yaml
```

### 3. Deploy the Workload and Service

Deploy a Pod to run the script and a Service to expose it to Prometheus:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: duplicate-metrics-app
  namespace: prometheus-demo-ns
  labels:
    app: duplicate-metrics
spec:
  replicas: 1
  selector:
    matchLabels:
      app: duplicate-metrics
  template:
    metadata:
      labels:
        app: duplicate-metrics
    spec:
      containers:
      - name: server
        image: python:3.9-slim
        command: ["python", "/script/server.py"]
        ports:
        - containerPort: 8080
          name: metrics
        volumeMounts:
        - name: script-volume
          mountPath: /script
      volumes:
      - name: script-volume
        configMap:
          name: duplicate-metrics-script
---
apiVersion: v1
kind: Service
metadata:
  name: duplicate-metrics-service
  namespace: prometheus-demo-ns
  labels:
    app: duplicate-metrics
spec:
  ports:
  - port: 8080
    targetPort: metrics
    name: metrics
  selector:
    app: duplicate-metrics
```

Apply the Deployment and Service:

```bash
oc apply -f duplicate-metrics-deployment.yaml
```

### 4. Create the ServiceMonitor

Create a `ServiceMonitor` to instruct the Prometheus operator to scrape this faulty endpoint:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: duplicate-metrics-monitor
  namespace: prometheus-demo-ns
  labels:
    app: duplicate-metrics
spec:
  selector:
    matchLabels:
      app: duplicate-metrics
  endpoints:
  - port: metrics
    interval: 15s
```

Apply the ServiceMonitor:

```bash
oc apply -f duplicate-metrics-servicemonitor.yaml
```

### 5. Trigger and Verify the Alert

Once the resources are applied, Prometheus will begin scraping the pod every 15 seconds.

1. **Verify the Metric Increase:** Query Prometheus (e.g., via the OpenShift Observe UI) for the following expression. You will see a value greater than `0`.

   ```promql
   rate(prometheus_target_scrapes_sample_duplicate_timestamp_total[5m])
   ```

2. **Check the Logs:** Run the diagnosis command on your Prometheus pod to see the ingestion errors:

   ```bash
   oc -n openshift-user-workload-monitoring logs -l 'app.kubernetes.io/name=prometheus' | grep 'Error on ingesting samples with different value but same timestamp'
   ```

   You should see logs indicating the target `duplicate-metrics-monitor` is throwing ingestion errors.

3. **Wait for Alert:** Because the expression `rate(prometheus_target_scrapes_sample_duplicate_timestamp_total...[5m]) > 0` evaluates to true, the `PrometheusDuplicateTimestamps` alert will fire after meeting its `for: 1h` duration threshold.

## Cleanup

### Delete the resources

To remove the reproducer environment and stop the alert, delete the created resources and the custom namespace:

```bash
oc delete servicemonitor duplicate-metrics-monitor -n prometheus-demo-ns
oc delete service duplicate-metrics-service -n prometheus-demo-ns
oc delete deployment duplicate-metrics-app -n prometheus-demo-ns
oc delete configmap duplicate-metrics-script -n prometheus-demo-ns

# Finally, delete the custom namespace
oc delete namespace prometheus-demo-ns
```
