# Reproduce ImageRegistryStorageFull Alert

This procedure reproduces the `ImageRegistryStorageFull` alert by configuring the image registry to use a temporary `1Gi` PVC, filling the PVC, and attempting to push an image to the full registry.

> **Warning:** Perform this procedure only on a test cluster. Image pushes to the registry will fail while the test PVC is full.

## 1. Save the Original Registry PVC

```bash
export ORIGINAL_PVC=$(oc get configs.imageregistry.operator.openshift.io cluster \
  -o jsonpath='{.spec.storage.pvc.claim}')

echo "$ORIGINAL_PVC"
```

## 2. Create a Temporary 1Gi PVC

```bash
cat <<EOF | oc apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-registry-test
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

Wait for the PVC to become `Bound`:

```bash
oc get pvc pvc-registry-test -n openshift-image-registry
```

## 3. Configure the Registry to Use the Temporary PVC

```bash
oc patch configs.imageregistry.operator.openshift.io cluster \
  --type merge \
  --patch '{"spec":{"storage":{"pvc":{"claim":"pvc-registry-test"}}}}'
```

Wait for the registry pod to become `Running`:

```bash
oc get pods -n openshift-image-registry \
  -l docker-registry=default -w
```

## 4. Fill the PVC

```bash
export POD_NAME=$(oc get pods \
  -n openshift-image-registry \
  -l docker-registry=default \
  -o jsonpath='{.items[0].metadata.name}')

oc exec -n openshift-image-registry "$POD_NAME" -- \
  fallocate -l 1G /registry/dummy-fill.img
```

## 5. Force an Image Push Failure

Expose the registry route:

```bash
oc patch configs.imageregistry.operator.openshift.io/cluster \
  --type=merge \
  --patch '{"spec":{"defaultRoute":true}}'

export HOST=$(oc get route default-route \
  -n openshift-image-registry \
  -o jsonpath='{.spec.host}')
```

Log in and push an image:

```bash
podman login \
  -u kubeadmin \
  -p "$(oc whoami -t)" \
  "$HOST" \
  --tls-verify=false

podman pull docker.io/library/alpine:latest

podman tag docker.io/library/alpine:latest \
  "$HOST/openshift/alpine-test:latest"

podman push "$HOST/openshift/alpine-test:latest" \
  --tls-verify=false
```

Expected failure example or similar failed error:

```text
Error writing blob: error storing blob to file ... : no space left on device
```

## 6. Validate the Alert

Wait approximately **5 minutes**, then go to:

**OpenShift Console → Observe → Alerting**

Verify that `ImageRegistryStorageFull` is firing.

## 7. Restore the Original Registry Storage

```bash
oc patch configs.imageregistry.operator.openshift.io cluster \
  --type merge \
  --patch "{\"spec\":{\"storage\":{\"pvc\":{\"claim\":\"$ORIGINAL_PVC\"}}}}"
```

Wait for the registry pod to become `Running`:

```bash
oc get pods -n openshift-image-registry \
  -l docker-registry=default -w
```

## 8. Clean Up

```bash
oc delete pvc pvc-registry-test \
  -n openshift-image-registry
```

Verify the registry:

```bash
oc get clusteroperator image-registry
```

The `AVAILABLE` condition should be `True`.
