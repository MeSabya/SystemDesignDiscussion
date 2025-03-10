

![image](https://github.com/user-attachments/assets/4cd373c5-a75b-467d-be12-dc178498ded7)

## Functional Requirements (FRs)
Defines what the system should do.

- Endpoint Registration – The system should allow defining endpoints to be monitored (via CRDs).
- Health Check Execution – The system should run periodic checks on each endpoint.
- Result Collection & Processing – Store health check results for analysis.
- Auto-Scaling Mechanism – Scale monitoring jobs based on failures.
- Alerting & Monitoring – Notify on failures via Prometheus alerts.
- Observability & Logs – Store logs for debugging and analytics.

## 2️⃣ Non-Functional Requirements (NFRs)
Defines how the system should perform.

### 🔹 Scalability

✔ Elastic Scaling – KEDA dynamically increases/decreases Jobs based on failures.

✔ Distributed Processing – Kafka allows multiple consumers to process results.

✔ Multi-Cluster Support – Can run across multiple Kubernetes clusters.

### 🔹 Reliability & Fault Tolerance

✔ High Availability – Multiple replicas of Operator, Kafka, and Prometheus prevent single points of failure.

✔ Auto-Retry Mechanism – Failed Jobs can be retried based on Kubernetes Job policies.

✔ Message Persistence – Kafka retains messages until consumed.

### 🔹 Performance
✔ Low Latency – Jobs execute quickly & push results in real-time.

✔ Efficient Resource Usage – Kubernetes ensures optimal pod scheduling.

✔ Parallel Execution – Multiple Jobs run concurrently for fast monitoring.

### 🔹 Observability & Monitoring

✔ Logs & Tracing – Health check results are logged and traceable.

✔ Dashboards – Grafana visualizes system health metrics.

✔ Alerting System – Prometheus triggers alerts on failures.

## Execution Flow (Step-by-Step)

### 1️⃣ User applies a HealthCheckConfig CRD

Defines endpoints, frequency, timeout, and failure thresholds.
⬇

### 2️⃣ Operator monitors the CRD and schedules a Kubernetes Job

The Operator reads the CRD and spawns a Job to perform the health check.
Jobs are short-lived and ensure isolated execution.
⬇

### 3️⃣ Job runs a Pod that monitors the endpoint

It performs health checks based on the CRD config (e.g., HTTP, TCP, gRPC).
If the endpoint is unhealthy, retries occur as per the CRD.
In Kubernetes, Jobs create Pods, but a Job itself is not a Pod. Here's how it works:

A Job is a higher-level Kubernetes resource that ensures a specified number of Pods complete successfully.
When you create a Job, it creates and manages one or more Pods to perform the given task.
If a Pod in a Job fails, the Job may create a new Pod to retry the task, depending on the restart policy.
⬇

### 4️⃣ Pod publishes the result to a Kafka topic

"success", "failure", or "timeout" messages are sent to Kafka (healthcheck-results).
⬇

### 5️⃣ Kafka Exporter exposes Kafka metrics for Prometheus

Kafka Exporter scrapes queue lag, pending messages, and throughput.
Metrics like kafka_topic_partition_current_offset and kafka_consumergroup_lag are exposed.
⬇

### 6️⃣ Prometheus scrapes metrics from Kafka Exporter

Tracks job failures, latency, and unprocessed messages.
Alerts on anomalies (e.g., high failure rate).
⬇

### 7️⃣ KEDA scales the Operator based on Prometheus metrics

KEDA is watching Prometheus health check failure rate (or other metrics).
✅ If too many failed health checks, KEDA scales Jobs to perform more frequent checks.
