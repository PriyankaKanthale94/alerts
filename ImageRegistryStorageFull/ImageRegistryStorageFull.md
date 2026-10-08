# ImageRegistryStorageFull

**PrometheusRule Source:** `Platform` · **Pending For:** `10m` · **Severity:** `Warning`

ImageRegistryStorageFull means that the image registry storage disk is full. A full disk affects direct pushes to the image registry and pull-through proxy caching. In the case of pull-through proxy caching, disk space is particularly important because without it the image registry will not be actually caching anything. 

**Expression**

```promql
sum without (instance, pod, operation) (
  rate(imageregistry_storage_errors_total{code="DEVICE_OUT_OF_SPACE"}[5m])
) > 0
```

## Impact

- Direct image pushes to the image registry will fail.
- Builds that push images to the registry may fail.
- Pull-through proxy caching cannot store images locally.
- Images imported through Image Streams or `oc import-image` may continue to be served from the upstream registry, but local caching will fail.

## Diagnosis

### 1. Check the image registry pods

```bash
oc get pods -n openshift-image-registry -l docker-registry=default
```

Check the registry pod logs for storage errors:

```bash
oc logs -n openshift-image-registry \
  -l docker-registry=default | grep -i -E 'no space|out.of.space|DEVICE_OUT_OF_SPACE'
```

### 2. Check the registry PVC

Identify the PVC configured for the registry:

```bash
oc get configs.imageregistry.operator.openshift.io cluster \
  -o jsonpath='{.spec.storage.pvc.claim}{"\n"}'
```

Check the PVC:

```bash
oc get pvc -n openshift-image-registry
```

Verify its capacity and status.

### 3. Check filesystem usage

Get the registry pod:

```bash
export POD_NAME=$(oc get pods \
  -n openshift-image-registry \
  -l docker-registry=default \
  -o jsonpath='{.items[0].metadata.name}')
```

Check the registry filesystem:

```bash
oc exec -n openshift-image-registry "$POD_NAME" -- df -h /registry
```

If the filesystem is at or near `100%`, the storage capacity is exhausted.


## Mitigation

### 1. Review image pruning

Check the [image-pruner](https://docs.redhat.com/en/documentation/openshift_container_platform/4.21/html/building_applications/pruning-objects#pruning-images_pruning-objects) configuration:

```bash
oc get imagepruners.imageregistry.operator.openshift.io -A
```

Review the configured pruning schedule and retention settings.

If image pruning is correctly configured but the registry continues to run out of space, additional storage may be required.

### 2. Expand the registry PVC as per the change management process.

Check the current PVC:

```bash
oc get pvc -n openshift-image-registry
```

If the storage class supports expansion, increase the PVC size:

```bash
oc edit pvc <registry-pvc> -n openshift-image-registry
```

Update:

```yaml
spec:
  resources:
    requests:
      storage: <new-size>
```

Verify the expanded capacity:

```bash
oc get pvc <registry-pvc> -n openshift-image-registry
```


### 3. Verify the backing storage

If the PVC has sufficient configured capacity but the filesystem remains full or the registry continues reporting `DEVICE_OUT_OF_SPACE`, check the underlying storage backend for:

- Storage quota exhaustion
- Backend capacity exhaustion
- Volume expansion failures
- Storage platform errors

Resolve the storage backend issue according to the storage provider's procedures.


## Resolution

After freeing or adding storage, verify:

```bash
oc exec -n openshift-image-registry "$POD_NAME" -- df -h /registry
```

## Verify
