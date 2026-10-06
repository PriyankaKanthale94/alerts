# Reproduce AuditLogError Alert

This procedure reproduces the `AuditLogError` alert by intentionally causing the Kubernetes API server to fail when writing to its audit log. By symlinking the active audit log file to `/dev/full`, we simulate a `No space left on device` error.

> **Note:** This procedure intentionally prevents audit events from being written on the affected API server. Run it only in an isolated test environment and never in production.

# Procedure

## 1. Access a Master Node

Identify the control-plane nodes:

```bash
oc get nodes -l node-role.kubernetes.io/master -o name
```

Pick one node and start a debug shell:

```bash
oc debug <master-node-name>
chroot /host
```

## 2. Simulate Disk Exhaustion (`/dev/full`)

Navigate to the API server audit-log directory, back up the active audit log, and replace it with a symlink to `/dev/full`:

```bash
cd /var/log/kube-apiserver
mv audit.log audit.log.bak
ln -s /dev/full audit.log
```

`/dev/full` causes write operations to fail with `ENOSPC` (`No space left on device`).

## 3. Restart the API Server Container

The API server must reopen the audit log for the `/dev/full` symlink to take effect. Stop only the main `kube-apiserver` container:

```bash
crictl ps --name "^kube-apiserver$" -q | xargs -r crictl stop
```

The kubelet will automatically recreate the container.

Wait for the API server to become ready:

```bash
oc get pods -n openshift-kube-apiserver -w
```

## 4. Generate API Traffic

Open a second terminal authenticated as a cluster administrator and generate continuous API requests:

```bash
while true
do
    oc get pods -A > /dev/null 2>&1
    echo -n "."
    sleep 0.1
done
```

## 5. Monitor the Alert 

### Using the OpenShift Console

Go to **Observe → Metrics → AuditLogError ** 

# Cleanup

## 1. Stop the Traffic Generator

Return to the second terminal and press:

```text
Ctrl+C
```

## 2. Restore the Original Audit Log

Return to the node debug terminal:

```bash
cd /var/log/kube-apiserver
rm audit.log
mv audit.log.bak audit.log
```

## 3. Restart the API Server Container

Force the API server to reopen the restored audit log:

```bash
crictl ps --name "^kube-apiserver$" -q | xargs -r crictl stop
```

Wait for the API server to become ready:

```bash
oc get pods -n openshift-kube-apiserver -w
```

Verify that `AuditLogError` clears and audit logging resumes normally.
