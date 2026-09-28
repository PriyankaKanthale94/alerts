# AlertmanagerReceiversNotConfigured

**PrometheusRule Source:** `Platform` · **Pending For:** `10m` · **Severity:** `Warning` · 

---

## Meaning

This alert indicates that alerts are not configured to be sent to a notification system. As a result, administrators may not be notified in a timely manner when important cluster failures occur.

By default, newly deployed OpenShift clusters may have Alertmanager provisioned without outbound notification integrations such as Slack, PagerDuty, email, or webhooks.

---

## Impact

* Administrators may not receive notifications for critical cluster failures or performance issues.
* Important alerts may only be visible through the OpenShift Web Console or CLI.
* Incident detection and response may be delayed.

---

## Diagnosis


### Inspect the Current Alertmanager Configuration

Review the current Alertmanager configuration:

```bash
oc -n openshift-monitoring get secret alertmanager-main \
  --template='{{ index .data "alertmanager.yaml" }}' \
  | base64 --decode
```

Review the `receivers:` section. Look for integration configuration such as:

```yaml
slack_configs:
webhook_configs:
email_configs:
pagerduty_configs:
```

---

## Mitigation

### Option 1: Configure a Real Notification Integration

If you want to receive external notifications, add a valid notification configuration to a receiver and ensure that the receiver is referenced in the Alertmanager route tree.
Check [OpenShift documentation](https://docs.redhat.com/en/documentation/openshift_container_platform/4.22/html/postinstallation_configuration/configuring-alert-notifications) 

### Option 2: Disable the Alert Without a Silence

If this is a non-production cluster or you intentionally do not want external notifications,[Red Hat KCS 6376561](https://access.redhat.com/solutions/6376561) documents a workaround using a dummy webhook configuration to satisfy the Alertmanager integration check.

---

## Verification

### Verify the Alert State

From the OpenShift webconsole navigate to Observe -> Alerting

### Check Alertmanager Logs

Verify that Alertmanager successfully loaded the configuration:

```bash
oc logs -l app.kubernetes.io/name=alertmanager \
  -n openshift-monitoring \
  -c alertmanager \
  --tail=50
```

