# Reproduce ClusterOperatorFlapping Alert

This procedure reproduces the `ClusterOperatorFlapping` alert using a dummy `ClusterOperator` resource. It does not modify or impact any real cluster operators.

## Prerequisites

- `cluster-admin` privileges or equivalent permissions to create, update, and delete `ClusterOperator` resources and access Prometheus metrics.

## 1. Create the Dummy ClusterOperator

```bash
cat <<EOF | oc apply -f -
apiVersion: config.openshift.io/v1
kind: ClusterOperator
metadata:
  name: dummy-flapping-operator
spec: {}
EOF
```

Verify the resource:

```bash
oc get clusteroperator dummy-flapping-operator
```

## 2. Create the Flapping Script

The script continuously alternates the dummy ClusterOperator between healthy and degraded states every 35 seconds, generating repeated state changes for Prometheus to detect.

Create `flap-worker.sh`:

```bash
#!/bin/bash

OPERATOR_NAME="dummy-flapping-operator"

while true
do
  DATE=$(date -u +"%Y-%m-%dT%H:%M:%SZ")

  oc patch clusteroperator/${OPERATOR_NAME} \
    --type=merge \
    --subresource=status \
    -p "{
      \"status\": {
        \"conditions\": [
          {\"type\":\"Available\",\"status\":\"False\",\"reason\":\"DummyFlap\",\"message\":\"Forced Failure\",\"lastTransitionTime\":\"$DATE\"},
          {\"type\":\"Progressing\",\"status\":\"False\",\"reason\":\"DummyFlap\",\"message\":\"Forced Failure\",\"lastTransitionTime\":\"$DATE\"},
          {\"type\":\"Degraded\",\"status\":\"True\",\"reason\":\"DummyFlap\",\"message\":\"Forced Failure\",\"lastTransitionTime\":\"$DATE\"}
        ]
      }
    }" >/dev/null

  sleep 35

  DATE=$(date -u +"%Y-%m-%dT%H:%M:%SZ")

  oc patch clusteroperator/${OPERATOR_NAME} \
    --type=merge \
    --subresource=status \
    -p "{
      \"status\": {
        \"conditions\": [
          {\"type\":\"Available\",\"status\":\"True\",\"reason\":\"DummyFlap\",\"message\":\"Forced Recovery\",\"lastTransitionTime\":\"$DATE\"},
          {\"type\":\"Progressing\",\"status\":\"False\",\"reason\":\"DummyFlap\",\"message\":\"Forced Recovery\",\"lastTransitionTime\":\"$DATE\"},
          {\"type\":\"Degraded\",\"status\":\"False\",\"reason\":\"DummyFlap\",\"message\":\"Forced Recovery\",\"lastTransitionTime\":\"$DATE\"}
        ]
      }
    }" >/dev/null

  sleep 35
done
```

Make the script executable:

```bash
chmod +x flap-worker.sh
```

## 3. Start the Flapping Script

```bash
./flap-worker.sh
```

Leave the script running until the alert fires almost for 10-15 mins. 

## 4. Verify the Operator Status

In a separate terminal:

```bash
watch -n 5 'oc get clusteroperator dummy-flapping-operator'
```

The status should alternate between DEGRADED: TRUE/FALSE


## 5. Verify the Alert

Login to the Cluster Web Console, Navigate : Observe -> Alerting

## Cleanup

Stop the flapping script:

```bash
pkill -f flap-worker.sh
```

Delete the dummy `ClusterOperator`:

```bash
oc delete clusteroperator dummy-flapping-operator
```

Verify that it has been removed:

```bash
oc get clusteroperator dummy-flapping-operator
```
