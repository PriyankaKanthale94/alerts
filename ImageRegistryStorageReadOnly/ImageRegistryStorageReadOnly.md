# ImageRegistryStorageReadOnly

**PrometheusRule Source:** `image-registry-rules` · **Pending For:** `10m` · **Severity:** `Warning` . **[Runbook](https://github.com/openshift/runbooks/blob/master/alerts/cluster-image-registry-operator/ImageRegistryStorageReadOnly.md#meaning)**

## Meaning

This alert is triggered when the image registry pvc storage is read-only.
Read-only storage prevents image pushes and builds from committing images. Pull-through caching also fails to store images locally, although images may still be served from upstream registries.

## Impact

- Direct image pushes and builds that require registry writes fail.
- Pull-through proxy caching cannot store images locally.
- Images may become unavailable if the upstream registry is unavailable and they are not cached locally.

## Diagnosis

Set the namespace:

```bash
export NAMESPACE="openshift-image-registry"
```

### 1. Check registry pods and operator health

```bash
oc -n $NAMESPACE get pods -l docker-registry=default
oc get clusteroperator image-registry
oc get configs.imageregistry.operator.openshift.io cluster -o yaml
```

### 2. Check the operator read-only setting

```bash
oc get configs.imageregistry.operator.openshift.io cluster \
  -o jsonpath='{.spec.readOnly}{"\n"}'
```

A value of `true` indicates that read-only mode is explicitly enabled.

### 3. Inspect PVC and PV status

```bash
oc -n $NAMESPACE get pvc
```

Identify the PVC configured for registry storage and inspect its associated PV:

```bash
PVC=$(oc -n $NAMESPACE get configs.imageregistry.operator.openshift.io cluster \
  -o jsonpath='{.spec.storage.pvc.claim}')

oc -n $NAMESPACE describe pvc "$PVC"
oc describe pv "$(oc -n $NAMESPACE get pvc "$PVC" \
  -o jsonpath='{.spec.volumeName}')"
```

Check for capacity issues, volume errors, and storage backend problems.

### 4. Check available filesystem space

```bash
oc -n $NAMESPACE exec deploy/image-registry -- df -h /registry
```

### 5. Inspect volume mounts

```bash
oc -n $NAMESPACE describe pod -l docker-registry=default \
  | grep -A 8 -i "Mounts:"
```

Check whether the registry storage volume is mounted read-only.

### 6. Test write access

```bash
oc -n $NAMESPACE exec deploy/image-registry -- \
  sh -c 'touch /registry/.readonly-test && rm /registry/.readonly-test'
```

A failure may indicate read-only storage, filesystem permissions, or another storage problem. A successful test does not rule out intermittent or backend-specific write failures.

### 7. Review registry logs

```bash
oc -n $NAMESPACE logs deploy/image-registry --tail=100 \
  | grep -iE 'read-only|permission denied|EACCES|no space left'
```

## Mitigation

Apply the action corresponding to the identified cause.

| Scenario | What to look for | Action / Mitigation |
|---|---|---|
| **Operator configured in `readOnly` mode** | Run `oc get configs.imageregistry.operator.openshift.io cluster -o jsonpath='{.spec.readOnly}'`. Check whether the output is `true`. | Disable read-only mode if it is not intentional: `oc patch configs.imageregistry.operator.openshift.io cluster --type=merge -p '{"spec":{"readOnly":false}}'` |
| **PVC volume mounted in read-only mode** | Run `oc -n $NAMESPACE get pod -l docker-registry=default -o yaml`. Check the volume mounts for `readOnly: true`. | Inspect the registry configuration using `oc edit configs.imageregistry.operator.openshift.io cluster`. Correct the relevant configuration or storage issue and allow the operator to reconcile the Deployment. |
| **Storage volume full / space exhausted** | Run `oc -n $NAMESPACE exec deploy/image-registry -- df -h /registry` and `oc -n $NAMESPACE get pvc`. Check filesystem usage and PVC capacity. | Expand the PVC if the StorageClass and CSI driver support expansion: `oc -n $NAMESPACE patch pvc <pvc-name> -p '{"spec":{"resources":{"requests":{"storage":"<new-size>"}}}}'`. Alternatively, prune unused images where appropriate using `oc adm prune images`, following the supported image-pruning procedure. |
| **Underlying storage volume configured as read-only** | Run `oc describe pv <pv-name>` and inspect storage-related events. Check for mount errors, filesystem corruption, I/O errors, or storage-controller issues. | Identify the storage backend and resolve the underlying fault using the storage provider's documented recovery procedure. Do not force-remount or repair a mounted production volume without following the provider's guidance. |
| **Object-storage write permissions lost (S3/Azure/GCS)** | Run `oc -n $NAMESPACE logs deploy/image-registry --since=30m`. Look for `AccessDenied`, `403 Forbidden`, `PutObject`, or credential errors. Identify the configured backend using `oc get configs.imageregistry.operator.openshift.io cluster -o yaml`. | Verify the applicable cloud IAM policy, bucket or container permissions, credentials, and encryption-key permissions. Restore the required write permissions through the approved cloud administration process. 
| **Cause remains unknown** | Run `oc adm inspect ns/$NAMESPACE` and review events using `oc get events -n $NAMESPACE --sort-by='.lastTimestamp'`. | Collect an `oc adm must-gather`for further investigation. |


## Verification

1. **Verify the Image Registry Cluster Operator**

   ```bash
   oc get clusteroperator image-registry
   ```

   Confirm `AVAILABLE=True`, `PROGRESSING=False`, and `DEGRADED=False`.

2. **Verify registry pods**

   ```bash
   oc get pods -n openshift-image-registry -l docker-registry=default
   ```

   Confirm the registry pods are `Running` and ready.

3. **Verify storage write access**

   For PVC-backed storage, run:

   ```bash
   oc exec -n openshift-image-registry deploy/image-registry -- \
     sh -c 'touch /registry/.readonly-test && rm /registry/.readonly-test'
   ```

   Confirm the command succeeds without read-only filesystem or permission errors. For object-storage backends, verify write access with an approved image push test.

4. **Verify the alert has cleared**

   Query the following metric in **Observe → Metrics**:

   ```promql
   image_registry_storage_read_only
   ```

   Confirm the metric value is `0` and the `ImageRegistryStorageReadOnly` alert is no longer firing. Allow time for Prometheus to reevaluate the alert.
