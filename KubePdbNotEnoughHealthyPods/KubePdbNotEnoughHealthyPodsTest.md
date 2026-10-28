## Reproduce KubePdbNotEnoughHealthyPods Alert

This procedure reproduces the `KubePdbNotEnoughHealthyPods` alert (and by extension, `PodDisruptionBudgetLimit`) by intentionally setting a `PodDisruptionBudget` minimum availability requirement higher than the actual number of running pods.

The test uses a simple 1-replica Deployment in a custom namespace prefixed with `openshift-` (which is required by the alert's monitoring regex `namespace=~"(openshift-.*|kube-.*|default)"`) and applies a PDB requiring 2 replicas.

## Procedure:

### 1. Create the Project and Target Deployment

Create a dedicated project for the test. We must use an `openshift-` prefix so the Prometheus rule evaluates it:

```bash
oc new-project openshift-pdb-demo
```

Create a single-replica application in this namespace:

```bash
oc create deployment nginx-pdb-test --image=nginx --replicas=1 -n openshift-pdb-demo
```

### 2. Create the PodDisruptionBudget

Create the following `PodDisruptionBudget` configuration in a file named `pdb-test.yaml`. It demands a minimum of 2 available pods, deliberately exceeding the 1 running replica:

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: nginx-pdb
  namespace: openshift-pdb-demo
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: nginx-pdb-test
```

Apply the configuration:

```bash
oc apply -f pdb-test.yaml
```

### 3. Verify the Configuration Mismatch

Confirm that the PDB was created and check its current status:

```bash
oc get pdb nginx-pdb -n openshift-pdb-demo
```

Expected result should show `CURRENT` (1) is less than `MIN AVAILABLE` (2) or `DESIRED` (2).

### 4. Monitor the Alert and Metrics

Keep monitoring the cluster and observe the behavior of the PDB metrics. The alert evaluates the condition for 15 minutes (`For: 15m`) before moving from `Pending` to `Firing`.

** Login to OpenShift webconsole -> Observe > Metrics **


## Cleanup

### Delete the Project

Deleting the project will automatically clean up the Deployment, the PodDisruptionBudget, and the running pods. 

```bash
oc delete project openshift-pdb-demo
```

After cleanup, verify that the project has been removed and the alert is resolving:

```bash
oc get project openshift-pdb-demo
```
