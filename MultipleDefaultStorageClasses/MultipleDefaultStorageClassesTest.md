# Reproduce MultipleDefaultStorageClasses Alert

This procedure reproduces the `MultipleDefaultStorageClasses` alert by intentionally
configuring more than one `StorageClass` as the default in the cluster.

The test creates two non-functional "dummy" storage classes using the `kubernetes.io/no-provisioner` 
provisioner. Both are annotated with `storageclass.kubernetes.io/is-default-class: "true"`. 
Since they use a dummy provisioner, they will not interfere with existing dynamic 
storage routing, but they will trigger the Cluster Storage Operator's alert after 
the 10-minute evaluation period.

# Procedure:

## 1. Create the Dummy Storage Classes

Create two storage classes, both annotated as the default:

```bash
cat <<EOF | oc apply -f -
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: dummy-default-a
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: dummy-default-b
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
EOF
```

## 2. Verify the Storage Classes

Confirm that the storage classes were created successfully and check that multiple classes are marked as `(default)`:

```bash
oc get storageclass
```

Expected result:
```text
NAME                        PROVISIONER                    RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
dummy-default-a (default)   kubernetes.io/no-provisioner   Delete          WaitForFirstConsumer   false                  10s
dummy-default-b (default)   kubernetes.io/no-provisioner   Delete          WaitForFirstConsumer   false                  10s
...
```

## 3. Monitor the Alert Status

Monitor the cluster to observe the default storage class count and the state of the alert. Note that the alert requires the condition to persist for 5 minutes to become `Pending`, and an additional 10 minutes to become `Firing` (15 minutes total).

**Observe > Metrics**


# Cleanup

## 1. Delete the Dummy Storage Classes

Remove the test storage classes to restore the cluster to a healthy state:

```bash
oc delete storageclass dummy-default-a dummy-default-b
```

## 2. Verify Deletion

After cleanup, verify that the dummy storage classes have been removed and only your original default storage class (if any) remains marked as `(default)`:

```bash
oc get storageclass
```

Once the count drops to 1 (or 0), the `MultipleDefaultStorageClasses` alert will automatically resolve.
