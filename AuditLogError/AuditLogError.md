# AuditLogError

**Alert Severity:** `Warning` · **PrometheusRule Source:** `Platform` · **Pending For:** `1m`. **[Runbook](https://github.com/openshift/runbooks/blob/master/alerts/cluster-kube-apiserver-operator/AuditLogError.md)

## Meaning

This alert indicates that an API Server instance is experiencing errors while writing audit logs.

The alert fires when the audit-log error rate is greater than zero relative to the audit event rate:

```promql
sum by (apiserver, instance) (
  rate(apiserver_audit_error_total{apiserver=~".+-apiserver"}[5m])
)
/
sum by (apiserver, instance) (
  rate(apiserver_audit_event_total{apiserver=~".+-apiserver"}[5m])
) > 0
```

## Impact

- Audit events may not be recorded by the affected API Server instance.

- Missing audit events can make it difficult to determine the impact of a security incident.

This alert does not by itself indicate that cluster availability is affected.

## Diagnosis

### 1. Identify the Affected API Server Instance

Query the audit error metric:

```promql
apiserver_audit_error_total{apiserver=~".+-apiserver"} > 0
```

Use the `apiserver` and `instance` labels to identify the affected API Server and node.



### 2. Check for Correlated Alerts

Check for other alerts that may indicate the underlying cause, such as `NodeFilesystemFillingUp`.

In the OpenShift Web Console, go to **Observe → Alerting** and check for active storage or node-related alerts affecting the node hosting the API Server instance.



### 3. Check API Server Runtime Logs

Identify the API Server pod running on the affected node:

```bash
oc get pods -n openshift-kube-apiserver -o wide | grep <affected-instance-ip>
```

Check the API Server logs:

```bash
oc logs -n openshift-kube-apiserver <api-server-pod-name> | \
grep -iE "audit|error"
```

Look for audit-log, filesystem, permission, or other write-related errors.

### 4. Verify Audit Log File Permissions and Attributes

Access the affected node:

```bash
oc debug node/<affected-node-name>
chroot /host
```

Verify the audit log ownership and permissions:

```bash
stat -c "Owner: %U, Permissions: %a, Path: %n" \
/var/log/kube-apiserver/audit*.log
```

Audit log files should be owned by `root` with permissions `0600`.

Check for unexpected filesystem attributes:

```bash
lsattr /var/log/kube-apiserver/audit*.log
```

Pay particular attention to unexpected `immutable` (`i`) or `append-only` (`a`) attributes.

### 5. Verify Audit Log Directory Permissions

```bash
stat -c "Owner: %U, Permissions: %a, Path: %n" \
/var/log/kube-apiserver
```

The audit log directory should be owned by `root` with permissions `0700`.

### 6. Escalate if Necessary

If audit log tampering is suspected, preserve the relevant evidence and follow the organization's security incident response procedure.

## Mitigation

Mitigation depends on the identified root cause and the organization's security and compliance requirements.

| Cause                            | Mitigation                                                                                           |
| -------------------------------- | ---------------------------------------------------------------------------------------------------- |
| Node filesystem is full          | Free sufficient disk space and investigate the cause of the disk consumption.                        |
| Incorrect audit log permissions  | Restore the expected ownership and permissions after confirming that the change was not intentional. |
| Unexpected filesystem attributes | Remove unauthorized `immutable` or `append-only` attributes after investigating the cause.           |
| API Server audit logging error   | Review API Server logs and resolve the underlying error.                                             |
| Suspected audit-log tampering    | Preserve evidence and follow the organization's incident response procedure.                         |
| Unknown cause                    | Escalate for further investigation and review the affected node and API Server logs by Capturing must-gather and the node sos-report for further investigation.               |

For RCA do not remove/restart a node unless required logs are captured.
