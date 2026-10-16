# Reproduce KubePersistentVolumeErrors Alert

This procedure reproduces the `KubePersistentVolumeErrors` warning alert by provisioning a `PersistentVolume` (PV) and `PersistentVolumeClaim` (PVC), releasing the volume, and patching its status subresource directly to `Failed`.

> [!WARNING] **Operational Impact & Alert Routing**
>
> Run this test only in non-production environments or during scheduled maintenance windows.
> - **Cluster Privileges:** `PersistentVolume` is a cluster-scoped object. You must have `cluster-admin` privileges (or equivalent RBAC permissions to create, patch, and delete PVs and query `openshift-monitoring`).
> - **Namespace Restriction:** The alert rule evaluates `namespace=~"(openshift-.*|kube-.*|default)"`. The claim must reside in a matching namespace (such as `default`) so `kube-state-metrics` reports the correct namespace label.

> **Limitation:** This procedure synthetically sets the PV status to `Failed`. It validates the metric and alerting pipeline but does not reproduce a genuine storage or volume-reclamation failure.

### Step 1: Create the PV and Bound PVC

Apply a PV configured with `persistentVolumeReclaimPolicy: Retain` and a matching PVC in the `default` namespace:

```bash
cat <<EOF | oc apply -f -
apiVersion: v1
kind: PersistentVolume
metadata:
  name: alert-test-pv
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: manual
  hostPath:
    path: /tmp/alert-test-dir
  claimRef:
    namespace: default
    name: alert-test-pvc
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: alert-test-pvc
  namespace: default
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: manual
  volumeName: alert-test-pv
  resources:
    requests:
      storage: 1Gi
EOF
```

### Step 2: Verify the Bound Status

Confirm that the PV and PVC bound successfully:

```bash
oc get pv alert-test-pv
```

Expected output:

```text
NAME            CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                    STORAGECLASS   AGE
alert-test-pv   1Gi        RWO            Retain           Bound    default/alert-test-pvc   manual         10s
```

### Step 3: Delete the PVC (Move PV to `Released`)

Delete the PVC so the PV transitions to the `Released` state:

```bash
oc delete pvc alert-test-pvc -n default
```

> **WARNING — Controller Reversion:** Do not skip deleting the PVC. If you attempt to patch a PV that is still actively bound to an existing PVC, the Kubernetes `persistentvolume-binder-controller` may reconcile its status back to `Bound`.

Verify that the PV is in `Released` status:

```bash
oc get pv alert-test-pv
```

Expected output:

```text
NAME            CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS     CLAIM                    STORAGECLASS   AGE
alert-test-pv   1Gi        RWO            Retain           Released   default/alert-test-pvc   manual         25s
```

The PV is now released from the active PVC, while its `claimRef.namespace: default` remains intact for metric matching.

### Step 4: Patch the Status Subresource to `Failed`

Execute a status subresource merge patch:

```bash
oc patch pv alert-test-pv \
  --subresource=status \
  --type=merge \
  -p '{"status":{"phase":"Failed"}}'
```

Confirm that the status is now `Failed`:

```bash
oc get pv alert-test-pv
```

Expected output:

```text
NAME            CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                    STORAGECLASS   AGE
alert-test-pv   1Gi        RWO            Retain           Failed   default/alert-test-pvc   manual         40s
```

Because there is no active PVC, the controller may be less likely to restore the previous phase. However, the synthetic status may still be rejected or reconciled by the API or controllers.

### Step 5: Monitor the Metric and Alert Status

#### OpenShift Web Console

1. Navigate to **Observe → Alerting**:
   - **Minutes 0–5:** The alert `KubePersistentVolumeErrors` should show in the **Pending** state if the expression matches.
   - **After 5 continuous minutes:** The alert should transition to **Firing** with severity **Warning**, provided the condition remains true.

## Step 6: Cleanup

> [!WARNING] **Avoid Alert Drift:** Remove the test PV promptly once verification is complete; otherwise, the alert may remain active until the metric series disappears and Prometheus evaluates the cleared condition.

### 1. Delete the PersistentVolume

```bash
oc delete pv alert-test-pv --ignore-not-found
```

### 2. Remove Finalizers (If Needed)

If the PV remains in the `Terminating` state, inspect its finalizers and related events before taking further action. If a finalizer is confirmed to be blocking deletion and it is safe to remove, clear it:

```bash
oc patch pv alert-test-pv -p '{"metadata":{"finalizers":null}}'
```

### 3. Verify Removal

```bash
oc get pv alert-test-pv
```

Expected output:

```text
Error from server (NotFound): persistentvolumes "alert-test-pv" not found
```

Once the PV is deleted, `kube-state-metrics` should stop exposing its metric series. The alert should resolve after Prometheus evaluates the cleared condition, provided no other PVs match the alert expression.

