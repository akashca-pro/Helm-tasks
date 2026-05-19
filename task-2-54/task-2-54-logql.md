# Task 2.54: Custom Grafana Dashboard and Loki Log Querying

## Dashboard Focus

### Selected Focus

Namespace Resource Health Dashboard

### Rationale

Most pre-built Grafana dashboards provide cluster-wide metrics but do not give a single focused view of one namespace's health.

This dashboard combines:

* Running pod count
* CPU usage
* Memory usage
* Pod restart count
* Error logs

This helps quickly answer:

"Is a namespace healthy and if not, what is causing issues?"

---

# Part 1: LogQL Queries

## Query 1: All logs from monitoring namespace

### Query

```logql
{namespace="akash-c-a-monitoring"}
```

### What it does

Returns all log streams from every pod running in the monitoring namespace.

### Components

Stream selector:

```logql
{namespace="akash-c-a-monitoring"}
```

* namespace is a label
* monitoring is the label value
* selects all streams matching this label

### output

```text
2026-05-19 20:23:37.437	
2026-05-19T14:53:37.225000494Z stdout F logger=tsdb.loki endpoint=callResource pluginId=loki dsName=loki-1 dsUID=dfmjdy3lddv5sd uname=admin t=2026-05-19T14:53:37.224904514Z level=info msg="Response received from loki" status=ok statusCode=200 contentLength=47 duration=1.180339ms contentEncoding= stage=databaseRequest
```

---

## Query 2: Grafana pod logs only

### Query

```logql
{namespace="monitoring",pod=~".*grafana.*"}
```

### What it does

Shows logs from Grafana pods only.

### Components

pod=~".*grafana.*"

Regex operator:

```text
=~
```

Matches pod names containing:

```text
grafana
```

### output

```text
2026-05-19 20:56:13.349	
2026-05-19T15:26:13.133699538Z stdout F logger=tsdb.loki endpoint=callResource pluginId=loki dsName=loki-1 dsUID=dfmjdy3lddv5sd uname=admin t=2026-05-19T15:26:13.132057845Z level=info msg="Response received from loki" status=ok statusCode=200 contentLength=77 duration=4.601534ms contentEncoding= stage=databaseRequest
```

---

## Query 3: Error logs from all namespaces

### Query

```logql
{namespace=~".+"} |= "error"
```

### What it does

Searches every namespace and returns only lines containing:

```text
error
```

### Components

Stream selector:

```logql
{namespace=~".+"}
```

Filter:

```logql
|= "error"
```

### output

```text
	
2026-05-19T15:26:10.990769089Z stdout F logger=context userId=0 orgId=0 uname= t=2026-05-19T15:26:10.990663427Z level=info msg="Request Completed" method=GET path=/api/live/ws status=401 remote_addr=127.0.0.1 time_ms=0 duration=637.849µs size=105 referer= handler=/api/live/ws status_source=server errorReason=Unauthorized errorMessageID=session.token.rotate error="token needs to be rotated"
```

---

## Query 4: Error logs excluding 404

### Query

```logql
{namespace=~".+"} |= "error" != "404"
```

### What it does

Returns error messages while removing noisy 404 errors.

### Components

Include:

```logql
|= "error"
```

Exclude:

```logql
!= "404"
```

### output

```text
2026-05-19 21:00:11.926	
2026-05-19T15:30:11.756070425Z stdout F logger=provisioning.dashboard type=file name=sidecarProvider t=2026-05-19T15:30:11.755959653Z level=error msg="failed to save dashboard" file=/tmp/dashboards/prometheus.json error="A dashboard with the same uid already exists"
```

---

## Query 5: Log rate metric

### Query

```logql
sum(
rate(
{namespace=~".+"}[5m]
)
) by(namespace)
```

### What it does

Converts log streams into a metric representing:

Log lines per second

### Components

rate()

Calculates rate over:

```text
5 minutes
```

sum()

Groups results by namespace.

### output

```text
output is visual metric graph
```

---

## Query 6: JSON parsing query

### Query

```logql
{namespace="monitoring"} | json | level=~"error|warn"
```

### What it does

Parses JSON logs and filters logs whose level field is:

* error
* warn

### Example JSON log

```json
{
 "time":"2026-05-19",
 "level":"error",
 "message":"Database failed"
}
```

### output

```text
No logs found
```

If no JSON logs exist:

Document:

No JSON logs were present.

The json parser is useful when applications generate structured JSON logs.

---

# Stream Selector vs Filter Expression

## Stream selector

Used to select log streams based on labels.

Example:

```logql
{namespace="monitoring"}
```

Stream selectors are mandatory.

---

## Filter expression

Filters actual log lines after stream selection.

Example:

```logql
|= "error"
```

---

# Why Loki indexes labels only

Loki indexes labels instead of log content.

Advantages:

* Lower storage cost
* Lower memory usage
* Faster ingestion
* Simpler architecture

Trade-off:

Searching inside actual log content becomes slower than Elasticsearch because Elasticsearch indexes every word.

---
# Dashboard Panel Documentation

## Panel 1

### Panel Name

Running Pods

### Query Used

```promql
count(
  kube_pod_status_phase{
    namespace="$namespace",
    phase="Running"
  }
)
```

### Why I Chose This Panel

This panel provides an immediate view of workload health in the selected namespace.

### Operational Question Answered

"How many pods are currently running and healthy in this namespace?"

---

## Panel 2

### Panel Name

CPU Usage by Pod

### Query Used

```promql
sum(
  rate(
    container_cpu_usage_seconds_total{
      namespace="$namespace",
      container!=""
    }[5m]
  )
) by (pod)
```

### Why I Chose This Panel

CPU usage helps identify workloads that consume excessive compute resources and can reveal performance bottlenecks.

### Operational Question Answered

"Which pods are consuming CPU resources and are any showing unusually high CPU usage?"

---

## Panel 3

### Panel Name

Memory Usage by Pod

### Query Used

```promql
sum(
  container_memory_working_set_bytes{
    namespace="$namespace",
    container!=""
  }
) by (pod)
```

### Why I Chose This Panel

Memory usage helps identify pods with high memory consumption and can indicate memory leaks or resource pressure.

### Operational Question Answered

"Which pods are consuming large amounts of memory?"

---

## Panel 4

### Panel Name

Pod Restart Count

### Query Used

```promql
sum by (pod)(
  increase(
    kube_pod_container_status_restarts_total{
      namespace="$namespace"
    }[1h]
  )
)
```

### Why I Chose This Panel

Frequent restarts usually indicate application failures, crashes, failed health checks, or resource problems.

### Operational Question Answered

"Are pods crashing or restarting repeatedly?"

---

## Panel 5

### Panel Name

Error Logs

### Query Used

```logql
{namespace="$namespace"} |= "error"
```

### Why I Chose This Panel

Metrics show symptoms, but logs explain the reason behind those symptoms. Error logs help quickly identify application issues.

### Operational Question Answered

"What errors are currently occurring in the selected namespace?"
