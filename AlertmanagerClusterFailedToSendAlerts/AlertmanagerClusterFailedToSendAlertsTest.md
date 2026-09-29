# Reproduce `AlertmanagerClusterFailedToSendAlerts` Alert

This procedure reproduces the `AlertmanagerClusterFailedToSendAlerts` alert by intentionally causing the cluster's Alertmanager instances to fail to deliver notifications.

The test introduces a receiver configured with an unreachable URL. By explicitly routing the continuously firing built-in `Watchdog` alert to this receiver, notification attempts will time out, causing the failure rate to exceed the 1% threshold and triggering the alert.

## Procedure

### 1. Backup the Existing Configuration

```bash
oc -n openshift-monitoring get secret alertmanager-main \
  --template='{{ index .data "alertmanager.yaml" }}' \
  | base64 --decode > alertmanager-backup.yaml
```

### 2. Create the Failing Configuration

```bash
cp alertmanager-backup.yaml alertmanager-reproducer.yaml

```
This configuration includes the `failing-receiver` with an unreachable IP (`203.0.113.1`) and a route that directs `Watchdog` alerts to it:

```bash
cat <<EOF > alertmanager-reproducer.yaml
...
routes:
  # --- 1. ADD the missing ROUTE BLOCK parameters  ---
  - matchers:
    - alertname = Watchdog
    receiver: 'failing-receiver'
  # -------------------------------
receivers:
- name: 'default'
# --- 2. ADD THIS RECEIVER BLOCK ---
- name: 'failing-receiver'
  webhook_configs:
  - url: 'http://203.0.113.1:9999/timeout' # Deliberately unreachable IP
    send_resolved: true
# ----------------------------------
```

### 3. Apply the Broken Configuration

Replace the existing Alertmanager configuration with the failing configuration:

```bash
oc -n openshift-monitoring create secret generic alertmanager-main \
  --from-file=alertmanager.yaml=alertmanager-reproducer.yaml \
  --dry-run=client -o yaml | oc apply -f -
```

> **Note:** The Alertmanager operator will detect the Secret change and automatically reload the configuration.

### 4. Verify 

Navigate the OpenShift WebConsole Observe -> Alerting -> Check for AlertmanagerClusterFailedToSendAlerts

## Cleanup

### Restore the Original Configuration from the backup

```bash
oc -n openshift-monitoring create secret generic alertmanager-main \
  --from-file=alertmanager.yaml=alertmanager-backup.yaml \
  --dry-run=client -o yaml | oc apply -f -
```

Verify that the Alertmanager configuration has been restored and that the `AlertmanagerClusterFailedToSendAlerts` alert resolves.

### Delete Temporary Files

```bash
rm alertmanager-backup.yaml alertmanager-reproducer.yaml
```
