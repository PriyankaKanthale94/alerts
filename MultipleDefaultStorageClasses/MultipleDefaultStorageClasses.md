# MultipleDefaultStorageClasses

**PrometheusRule Source:** `cluster-storage-operator` · **Pending For:** `10m` · **Severity:** `Warning` · 
---
This alert indicates that the Cluster Storage Operator has detected more than one `StorageClass` configured as the default in the cluster. 

OpenShift allows administrators to define a default storage class using the `storageclass.kubernetes.io/is-default-class` annotation. If more than one storage class has this annotation set to `"true"` consistently for at least 10 minutes, the `MultipleDefaultStorageClasses` alert is fired.

The alert evaluates a single expression over a 5-minute rolling window, and if it exceeds the threshold for 10 minutes, it transitions to a firing state.


## Impact

- If a user creates a Persistent Volume Claim (PVC) without explicitly specifying a `storageClassName` in the spec section, the cluster will have to choose between the multiple defaults.
- The storage class chosen for the PVC will depend on a specific OpenShift internal algorithm, which may lead to PVCs being bound to unexpected, suboptimal, or overly expensive storage backends.
- Ambiguity in dynamic volume provisioning cluster-wide.

---

## Diagnosis

The alert is caused by multiple storage classes having the `storageclass.kubernetes.io/is-default-class: "true"` annotation.

***1. List Available Storage Classes***

Run the following command to check for multiple storage classes marked with `(default)`:

```bash
oc get storageclass
```

*Example Output:*
```text
NAME            PROVISIONER                                            RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
gp2 (default)   kubernetes.io/aws-ebs                                  Delete          WaitForFirstConsumer   true                   267d
gp3 (default)   ebs.csi.aws.com                                        Delete          WaitForFirstConsumer   true                   267d
nfs             cluster.local/nfs-provisioner-nfs-server-provisioner   Delete          Immediate              true                   180d
```

***2. Verify the Annotations***

Print the YAML output of the default storage classes identified in the previous step and check for the `storageclass.kubernetes.io/is-default-class` annotation:

```bash
oc get storageclass <storage-class-name> -o yaml | grep storageclass.kubernetes.io/is-default-class
```

*Example:*
```bash
oc get storageclass gp3 -o yaml | grep storageclass.kubernetes.io/is-default-class
    storageclass.kubernetes.io/is-default-class: "true"

oc get storageclass gp2 -o yaml | grep storageclass.kubernetes.io/is-default-class
    storageclass.kubernetes.io/is-default-class: "true"
```
---

## Mitigation

To avoid routing ambiguities, it is required to keep only a **single default storage class**.

***1. Identify the intended default StorageClass***

Review the storage classes marked as default and consult with your cluster administrators or application owners to determine which storage class should be the actual default (e.g., `gp3` instead of the legacy `gp2`).

***2. Remove the default annotation from unintended StorageClasses***

For **all** storage classes that you no longer need to keep as default, set the `storageclass.kubernetes.io/is-default-class` annotation to `"false"`. Note: You must execute this command for all extra default storage classes until only one remains.

```bash
oc patch storageclass <storage-class-name> -p '{"metadata": {"annotations": {"storageclass.kubernetes.io/is-default-class": "false"}}}'
```

*Example (removing default from gp2):*
```bash
oc patch storageclass gp2 -p '{"metadata": {"annotations": {"storageclass.kubernetes.io/is-default-class": "false"}}}'
```

***3. Verify Mitigation***

Run `oc get storageclass` again. Ensure that only one storage class displays the `(default)` suffix. 

```bash
oc get sc | grep default
```

Once only one storage class is marked as default, the `default_storage_class_count` metric will drop to `1`, and the alert will automatically resolve.
