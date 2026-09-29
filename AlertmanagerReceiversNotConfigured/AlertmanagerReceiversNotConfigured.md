# Reproduce AlertmanagerReceiversNotConfigured Alert

This procedure reproduces the `AlertmanagerReceiversNotConfigured` alert by intentionally modifying the OpenShift Alertmanager configuration to comment out all valid outbound notification configuration.

## Requirement

The cluster should have Alertmanager Receivers configured.  

Verify : 
```bash
# oc -n openshift-monitoring get secret alertmanager-main --template='{{ index .data "alertmanager.yaml" }}' | base64 --decode

example : 
....
 receivers:
  - name: Default
  - name: Watchdog
  - name: Critical
  - name: slack
    slack_configs:
      - channel: '#my-channel'
        api_url: 'https://slack.com'
....
```

## Procedure

### 1. Backup the Current Configuration

Create a backup of your existing Alertmanager configuration.

```bash
oc -n openshift-monitoring get secret alertmanager-main \
  --template='{{ index .data "alertmanager.yaml" }}' | base64 --decode > alertmanager-backup.yaml
```

### 2. Comment the receiver configuration in alertmanager-backup.yaml or remove the section and create another file with changed configs. 


```bash

receivers:
  - name: Default
  - name: Watchdog
  - name: Critical
  - name: slack
#    slack_configs:
#      - channel: '#my-channel'
#        api_url: 'https://slack.com'


```

### 3. Apply the Reproducer Configuration

```bash
oc create secret generic alertmanager-main \
  -n openshift-monitoring \
  --from-file=alertmanager.yaml=alertmanager-backup.yaml \
  --dry-run=client -o yaml | oc replace -f -
```

### 4. Monitor the Status of the Alert

Using the OpenShift Web Console -> Observe -> Alerting

Wait 1–2 minutes and check for `AlertmanagerReceiversNotConfigured`


## Cleanup

### 5. Restore the Valid Configuration

Remove the ``#`` from alertmanager-backup.yaml file. 


### 6. Apply the Cleanup Configuration

```bash
oc create secret generic alertmanager-main \
  -n openshift-monitoring \
  --from-file=alertmanager.yaml=alertmanager-backup.yaml \
  --dry-run=client -o yaml | oc replace -f -
```

### 7. Verify

Verify that the Alertmanager configuration has been reloaded successfully by checking the Alertmanager logs:

```bash
oc logs -l app.kubernetes.io/name=alertmanager \
  -n openshift-monitoring \
  -c alertmanager \
  --tail=10
```
Verify that the alert is no longer firing.


### 8. Remove Local Test Files

Delete the local YAML files used for the test:

```bash
rm alertmanager-backup.yaml
```
