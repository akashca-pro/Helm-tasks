# Contact Point

## Details

**Name:**
`task-2-55-contact-point`

**Type:**
Slack

**Configuration:**
Slack channel webhook added

**Verification:**
Test notification sent successfully and received.

---

# Alert Rule 1

## Configuration

**Name:**
`task-2-55-cpu-alert`

**Query:**

```promql
100 - (
avg(
rate(
node_cpu_seconds_total{
mode="idle"
}[5m]
)
) * 100
)
```

**Threshold:**
Above 80%

**Pending period:**
1 minute

**Evaluation interval:**
1 minute

## Labels

* `severity=warning`
* `team=platform`

## Annotations

**Summary:**

```text
High CPU usage on {{ $labels.instance }}
```

**Description:**

```text
CPU usage on {{ $labels.instance }} has been above 80% for more than 2 minutes.

Current value: {{ $values.A }}%
```

## Reason for 1 Minute Pending Period

Prevents false alerts from short CPU spikes.

---

# Alert Rule 2

## Configuration

**Name:**
`task-2-55-pod-restart-alert`

**Query:**

```promql
increase(
kube_pod_container_status_restarts_total[15m]
) > 3
```

**Threshold:**

IS ABOVE 0

**Pending period:**

0 minutes (fire immediately)

**Evaluation interval:**

1 minute

## Labels

* `severity=critical`
* `team=platform`

## Annotations

**Summary:**

```text
Pod {{ $labels.pod }} in {{ $labels.namespace }} is crash-looping
```

**Description:**

```text
Pod {{ $labels.pod }} has restarted {{ $values.A }} times in the last 15 minutes
```

## Reason for Immediate Firing

A pod entering a crash loop is usually an active failure that can affect application availability immediately. Unlike temporary CPU spikes, repeated restarts generally indicate a real issue such as application crashes, failed health checks, missing dependencies, or resource exhaustion.

## Difference from CPU Alert Pending Period

The CPU alert uses a 2-minute pending period because CPU spikes can happen briefly and recover naturally. The pod restart alert uses a 0-minute pending period because repeated pod restarts are more urgent and should be reported immediately.

---

# Notification Policy

## Routing Tree

```text
Root Policy
└── severity=critical
    └── task-2-55-contact-point
```

**Matching Label:**

`severity=critical`

**Contact Point:**

`task-2-55-contact-point`

**Group By:**

* namespace
* pod

**Group Wait:**

30 seconds

**Group Interval:**

5 minutes

**Repeat Interval:**

1 hour

## Explanation

### Group Wait

Time Grafana waits before sending the first notification so related alerts can be grouped.

### Group Interval

Minimum time before sending updates for an existing alert group.

### Repeat Interval

How often Grafana re-sends notifications while alerts remain firing.

## Scenario

If 10 pods inside the same namespace crash-loop simultaneously:

Instead of sending 10 separate notifications, Grafana groups alerts based on namespace and pod labels and reduces notification noise.

---

# Alert State Observation

## Contact Point Verification

**Contact Point:**

`task-2-55-contact-point`

**Type:**

Slack

**Verification:**

A test notification was sent successfully and received in the Slack channel.

**Message Received:**

```text
Hello from Grafana test
```

**Result:**

Contact point configuration works correctly and notifications are successfully delivered to Slack.

---

## task-2-55-cpu-alert

### Observed State

**Firing**

### Observed Alert

`DatasourceNoData`

### Labels

* `alertname=DatasourceNoData`
* `rulename=task-2-55-cpu-alert`
* `severity=warning`
* `team=platform`

### Reason

The PromQL query for the CPU alert returned no values during evaluation. Instead of evaluating CPU usage against the threshold, Grafana generated a DatasourceNoData alert.

### Observed Notification

```text
CPU usage on [no value] has been above 80% for more than 1 minute.
Current value: [no value]%
```

### Explanation

This does not indicate high CPU usage. It indicates that Grafana could not obtain metric data from Prometheus for the configured query.

### Possible Causes

* Incorrect metric name
* Missing node exporter metrics
* Wrong label selection
* Prometheus target unavailable
* Query returning empty results

### State Meaning

DatasourceNoData means the alert engine executed successfully, but the query itself returned no metric data.

---

## task-2-55-pod-restart-alert

### Observed State

**Normal**

### Reason

No pod restarted more than three times during the previous 15 minutes.

### Manual Query Verification

**Query:**

```promql
increase(
kube_pod_container_status_restarts_total[15m]
) > 3
```

**Result:**

No matching pods returned.

### Explanation

Since no pods crossed the configured restart threshold, the alert remained in Normal state.

---

## Slack Notifications Observed

### Notification 1

```text
[FIRING:1] TestAlert
```

**Purpose:**

Generated during contact point testing to confirm Slack integration.

**Result:**

Successfully received.

---

### Notification 2

```text
[FIRING:1] DatasourceNoData task-2-55-alerts
```

**Purpose:**

Generated automatically by Grafana because `task-2-55-cpu-alert` returned no metric data.

**Result:**

Successfully received in Slack.

**Observation:**

Slack notification routing and contact point configuration are functioning correctly.

# Silence Configuration

## Configuration

**Matchers:**

```text
team=platform
```

**Duration:**

```text
1 hour
```

**Comment:**

```text
Silencing platform team alerts for Task 2.55 testing
```

## Observation

Both alert rules were affected because both contain the label:

```text
team=platform
```

Notifications were suppressed while the underlying alert rules continued evaluating normally.

## Difference Between Silenced and Resolved Alert

### Silenced Alert

* Alert condition still exists
* Alert may still be firing internally
* Notifications are suppressed
* Alert remains visible in Grafana

### Resolved Alert

* Alert condition is no longer true
* Alert returns to Normal state
* Recovery notification may be sent

## Real Production Example

During Kubernetes worker node maintenance:

Suppose worker node `worker-1` is being patched and restarted.

Multiple alerts may trigger:

* Node unavailable
* Pod restarts
* High latency
* Failed health checks

A temporary silence can suppress these expected alerts for the maintenance period and avoid unnecessary notification noise.

# Inhibition Rule Design

## Scenario

Source Alert (Inhibitor):

```text
task-2-55-cpu-alert
severity=warning
```

Target Alert (Suppressed):

```text
task-2-55-pod-restart-alert
severity=critical
```

Rationale:

If CPU usage becomes extremely high on a node, pods on that node may restart because of resource pressure rather than an application bug. Suppressing downstream restart alerts reduces notification noise.

## Alertmanager Inhibition Rule

```yaml
inhibit_rules:
  - source_matchers:
      - alertname="task-2-55-cpu-alert"
      - severity="warning"

    target_matchers:
      - alertname="task-2-55-pod-restart-alert"

    equal:
      - instance
```

## Explanation of equal Field

The `equal` field determines which labels must have identical values between source and target alerts before suppression happens.

In this rule:

```yaml
equal:
  - instance
```

means:

Both alerts must come from the same node or instance.

Example:

CPU Alert:

```text
instance=node-1
```

Pod Restart Alert:

```text
instance=node-1
```

Result:

```text
Suppression occurs
```

If:

CPU Alert:

```text
instance=node-1
```

Pod Restart Alert:

```text
instance=node-2
```

Result:

```text
No suppression
```

Without the `equal` field, a CPU alert on one node could incorrectly suppress pod restart alerts on unrelated nodes.
