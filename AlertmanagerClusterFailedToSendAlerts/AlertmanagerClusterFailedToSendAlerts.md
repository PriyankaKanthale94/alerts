# AlertmanagerClusterFailedToSendAlerts

**PrometheusRule Source:** `` · **Pending For:** `5m` · **Severity:** `Warning` · [Runbook](https://github.com/openshift/runbooks/blob/master/alerts/cluster-monitoring-operator/AlertmanagerClusterFailedToSendAlerts.md)

---

This alert indicates that **all Alertmanager instances in the cluster** are consistently failing to send notifications to a specific integration.

The affected integration may be Slack, PagerDuty, webhook, email, or another configured notification endpoint.

i.e The minimum notification failure rate to `{{ $labels.integration }}` sent from any instance in the `{{ $labels.job }}` cluster is `{{ $value | humanizePercentage }}`.

## Impact

* Notifications are not delivered to the affected integration.
* Administrators or on-call personnel may not receive alerts through the affected notification channel.
* The failure affects all Alertmanager instances, so Alertmanager high availability does not provide a functioning instance for the affected integration.

## Diagnosis

***1. Check Alertmanager Pod Logs***

Review the logs of all `alertmanager-main` pods in the `openshift-monitoring` namespace:

```bash
oc -n openshift-monitoring logs -l 'alertmanager=main'
```

Look for errors related to notification delivery, including:

* Endpoint unreachable
* Connection timeout
* DNS resolution failure
* Incorrect endpoint URL
* Invalid credentials
* HTTP 4xx/5xx responses

***2. Identify the Affected Integration***

Review the active alert to identify the `integration` label using Observe -> Alerting 


***3. Verify the Notification Endpoint***

If the logs indicate that the endpoint is unreachable or a DNS/network problem is suspected, verify connectivity to the affected endpoint from the Alertmanager environment.

```bash
oc -n openshift-monitoring exec -it <alertmanager_pod> -c alertmanager -- /bin/sh
```

Then:

```bash
curl -v https://<external-service-url>
```

***4. Review the Alertmanager Configuration***

Verify that the affected integration has the correct endpoint and credentials:

```bash
oc -n openshift-monitoring get secret alertmanager-main \
  --template='{{ index .data "alertmanager.yaml" }}' \
  | base64 --decode
```

## Mitigation

***1. Resolve Endpoint or Network Issues***

If the endpoint is unreachable:

* Verify the external notification service is available.
* Check DNS resolution.
* Check egress firewall or NetworkPolicy configuration.
* Check proxy configuration if applicable.

***2. Fix Configuration Errors***

If the logs indicate an incorrect URL or credentials:

1. Identify the correct configuration for the Alertmanager.
2. Correct the endpoint, credentials, or other affected configuration.
3. Allow the monitoring stack to reconcile and reload the configuration.

> **Note:** Follow the change management process to make any changes in configuration of secret `alertmanager-main`

***3. Verify Recovery***

Confirm that `AlertmanagerClusterFailedToSendAlerts` resolves.
