# Reproduce ImageRegistryStorageReadOnly Alert

**PrometheusRule Source:** `openshift-monitoring` · **Pending For:** `10m` · **Severity:** `Warning`

> **Warning:** This procedure changes the image registry storage configuration to a temporary PVC and sets the registry operator to `Unmanaged`. Use only in a test cluster. Save the original configuration and complete the cleanup steps to restore the cluster.

## Procedure

### 1. Create a Temporary PVC

```bash
cat <<EOF | oc apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-registry-ro-test
  namespace: openshift-image-registry
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  storageClassName: ocs-external-storagecluster-ceph-rbd
EOF
```

### 2. Configure Registry Storage

```bash
oc patch configs.imageregistry.operator.openshift.io cluster \
  --type merge \
  --patch '{"spec":{"storage":{"pvc":{"claim":"pvc-registry-ro-test"}}}}'
```

Wait for the registry pods to restart and mount the temporary PVC.

### 3. Pause Operator Reconciliation

```bash
oc patch configs.imageregistry.operator.openshift.io cluster \
  --type merge \
  --patch '{"spec":{"managementState":"Unmanaged"}}'
```

### 4. Mount Storage as Read-Only

```bash
oc set volume deployment/image-registry \
  -n openshift-image-registry \
  --add --name=registry-storage \
  --mount-path=/registry \
  --read-only=true \
  --overwrite
```

Wait for the updated pod to become ready.

### 5. Generate Write Errors

Create `worker.sh`:

```bash
#!/bin/bash

oc patch configs.imageregistry.operator.openshift.io cluster \
  --type merge --patch '{"spec":{"defaultRoute":true}}'

HOST=$(oc get route default-route -n openshift-image-registry \
  -o jsonpath='{.spec.host}')

podman login -u kubeadmin -p "$(oc whoami -t)" "$HOST" \
  --tls-verify=false

podman pull docker.io/library/alpine:latest
podman tag docker.io/library/alpine:latest \
  "$HOST/openshift/alpine-test:latest"

while true; do
  podman push "$HOST/openshift/alpine-test:latest" \
    --tls-verify=false
  sleep 15
done
```

Run the worker:

```bash
chmod +x worker.sh
./worker.sh
```

### 6. Verify the Alert

In **Observe → Metrics**, run:

```promql
sum without (instance, pod, operation) (
  rate(imageregistry_storage_errors_total{
    code="READ_ONLY_FILESYSTEM"
  }[5m])
)
```

Confirm the metric is greater than zero and wait for the alert's 10-minute pending period.

## Cleanup

### 1. Stop the Worker

Stop the script using `Ctrl+C` if running in the foreground. If running in the background, terminate the specific worker process.

### 2. Restore Registry Configuration

Restore the original storage claim and set the operator back to `Managed`:

```bash
oc patch configs.imageregistry.operator.openshift.io cluster \
  --type merge \
  --patch '{"spec":{"managementState":"Managed","storage":{"pvc":{"claim":"pvc-image-registry"}}}}'
```

**Note:** Replace `pvc-image-registry` with the original claim name if different. Verify that the registry deployment no longer has the manual read-only mount override.

### 3. Delete the Temporary PVC

After confirming the registry has returned to its original storage configuration:

```bash
oc delete pvc pvc-registry-ro-test -n openshift-image-registry
```

### 4. Verify Recovery

```bash
oc get pods -n openshift-image-registry
oc get deployment image-registry -n openshift-image-registry
oc get pvc -n openshift-image-registry
```

Confirm the registry pods are ready and the registry is operating normally.
