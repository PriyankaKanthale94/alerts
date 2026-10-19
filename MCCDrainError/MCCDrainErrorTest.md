# Reproduce MCCDrainError Alert

This procedure reproduces the MCCDrainError alert by isolating the reproduction workload inside a dedicated OpenShift project (`drain-test`) and scheduling it onto a target worker node. A strict PodDisruptionBudget (PDB) is configured to block eviction, and a reboot-forcing MachineConfig is applied to trigger an active node drain cycle.

This scenario is representative of a production situation where workloads running in user or application namespaces enforce strict disruption budgets or get stuck during an OpenShift upgrade or node maintenance. The Machine Config Controller (MCC) will cordon the node and repeatedly retry eviction until hitting the 1-hour timeout threshold, which sets the machine degraded state and fires the MCCDrainError alert.

## 1. Create a Dedicated Project

Create a dedicated test project to keep all testing artifacts isolated from core cluster namespaces:

```bash
oc new-project drain-test
```

## 2. Identify a Worker Node and Deploy the Blocker Workload

Select one of the worker nodes and deploy the blocker workload with an active PodDisruptionBudget directly to that node:

```bash
# Capture the name of an active worker node
TARGET_NODE="$(oc get nodes -l node-role.kubernetes.io/worker -o jsonpath='{.items[0].metadata.name}')"

cat <<EOF | oc apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: drain-blocker
  namespace: drain-test
spec:
  replicas: 1
  selector:
    matchLabels:
      app: drain-blocker
  template:
    metadata:
      labels:
        app: drain-blocker
    spec:
      nodeName: ${TARGET_NODE}
      containers:
      - name: sleep
        image: registry.access.redhat.com/ubi8/ubi-minimal
        command: ["sleep", "infinity"]
---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: drain-blocker-pdb
  namespace: drain-test
spec:
  minAvailable: 1
  selector:
    matchLabels:
      app: drain-blocker
EOF
```

Verify that the pod is running on the intended worker node:

```bash
oc get pod -n drain-test -o wide
```

Ensure the disruption budget shows `ALLOWED DISRUPTIONS: 0`:

```bash
oc get pdb -n drain-test
```

## 3. Apply a MachineConfig Requiring a Node Reboot

Apply a configuration update requiring an operating system restart so the Machine Config Operator initiates a full cordon-and-drain procedure:

```bash
cat <<EOF | oc apply -f -
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfig
metadata:
  labels:
    machineconfiguration.openshift.io/role: worker
  name: 99-force-worker-drain
spec:
  kernelArguments:
    - dummy.drain.test=true
EOF
```

## 4. Verify Node Cordon and Drain Failure

Watch the target worker node transition to cordoned:

```bash
oc get nodes -l node-role.kubernetes.io/worker
```

Follow the machine-config-controller logs to confirm eviction rejections:

```bash
CONTROLLER_POD=$(oc get pods -n openshift-machine-config-operator -l k8s-app=machine-config-controller -o jsonpath='{.items[0].metadata.name}')

oc logs -n openshift-machine-config-operator $CONTROLLER_POD -c machine-config-controller -f | grep -E "initiating drain|Drain failed"
```

Sample output:

```text
drain_controller.go:173] node worker-0: initiating drain
drain_controller.go:173] node worker-0: Drain failed. Waiting 1 minute then retrying. Error message from drain: Cannot evict pod as it would violate the pod's disruption budget.
```

## 5. Wait for the 1-Hour Drain Timeout

The controller retries eviction using an internal backoff strategy (retrying every 1 minute, and after 10 minutes, retrying every 5 minutes). Maintain this state for 60 minutes.

After 1 hour has elapsed, the controller logs:

```text
drain_controller.go:352] node worker-0: drain exceeded timeout: 1h0m0s. Will continue to retry.
status.go:126] Degraded Machine: worker-0: and Degraded Reason: failed to drain node: worker-0 after 1 hour.
```

This triggers the Prometheus gauge `mcc_drain_err{exported_node="<worker-node>"}` to change from 0 to 1.

## 6. Validate the MCCDrainError Alert

Open the OpenShift Web Console and navigate to **Observe > Alerting**.

Verify that the alert MCCDrainError is actively firing with Severity: Warning. You can also verify the metric in Observe > Metrics:

```promql
mcc_drain_err > 0
```

## 7. Mitigate the Drain Failure

Remove the blocking PodDisruptionBudget in the `drain-test` project so the controller can evict the pod:

```bash
oc -n drain-test delete pdb drain-blocker-pdb
```

(Note: If troubleshooting a stuck pod or storage unmount in production, you can force delete the pod to bypass graceful deletion: `oc delete pod <pod-name> -n <namespace> --force=true --grace-period=0`.)

## 8. Verify Recovery and Clean Up

Watch the Machine Config Controller complete the drain, reboot the worker node, and update the MachineConfigPool:

```bash
oc get mcp worker -w
```

Confirm that the alert clears as `mcc_drain_err` returns to 0. Once the pool status shows UPDATED: True, delete the test resources:

```bash
oc delete machineconfig 99-force-worker-drain
oc delete project drain-test
```
