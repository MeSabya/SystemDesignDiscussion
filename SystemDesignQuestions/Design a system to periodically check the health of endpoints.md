## Step 1: Clarify Requirements (Staff-Level Depth)
Ask clarifying questions to demonstrate breadth of thought:

- What’s an endpoint? Microservice? External URL?
- How often should checks be done (e.g., every 5s, 1 min)?
- What kind of checks? Ping, HTTP 200, custom probes?
- How should failures be reported (dashboard? PagerDuty? Webhook?)
- Is the system multi-tenant (e.g., checking endpoints for multiple customers)?
- How many endpoints (100? 100k?) — scalability requirement?
- Should it support distributed/geo-redundant deployment?

## High-Level Architecture: Health Check System

```pgsql
                                ┌─────────────────────┐
                                │      External UI     │
                                └─────────┬────────────┘
                                          │
                                 [HTTP API Gateway]
                                          │
                                          ▼
                               ┌──────────────────────┐
                               │ Kafka Topic: endpoint│
                               │ config submissions   │
                               └────────┬─────────────┘
                                        │
                                        ▼
                    ┌──────────────────────────────────┐
                    │ Job ID Generator Service         │
                    │ - Consumes config submissions    |
                    │ - genearate Job ID based on      |
                    |    twitter snoflake id           |
                    │ - Assigns Job IDs                │
                    │ - Writes to Config DB            | 
                    └────────────────┬─────────────────┘
                                     ▼
                       ┌─────────────────────────────┐
                       │     Config DB (CP System)    │
                       │  - Stores endpoint config    │
                       │  - Partitioned by Job ID     │
                       └─────────────────────────────┘
                                     │
                              (Scheduled Polling)
                                     ▼
                          ┌────────────────────────┐
                          │  HA Job Scheduler (CP) │
                          │  - Reads Config DB     │
                          │  - Uses Leader Election│
                          │  - Produces to Kafka   │
                          └─────────┬──────────────┘
                                    ▼
                     ┌─────────────────────────────┐
                     │ Kafka Topic: scheduled-jobs │
                     └────────────┬────────────────┘
                                  ▼
                 ┌────────────────────────────────────────┐
                 │ Worker Pool (K8s Deployment + HPA)     │
                 │ - Kafka Consumers (consumer group)     │
                 │ - Perform Health Checks                │
                 │ - Push results                         │
                 └────────────┬───────────────────────────┘
                              ▼
             ┌────────────────────────────────┐
             │ Kafka Topic: health-check-results │
             └────────────────┬────────────────┘
                              ▼
                     ┌────────────────────┐
                     │  Results DB (AP DB)│
                     │  - Write heavy DB  
                           cassandra      │
                     │  - Append-only     │
                     └────────┬───────────┘
                              ▼
         ┌──────────────────────────────────────────┐
         │ Notification Dispatcher                  │
         │ - Consumes from results Kafka topic      │
         │ - Sends alerts via Email, Slack, etc.    │
         └────────────┬─────────────────────────────┘
                      ▼
         ┌──────────────────────────────────────────┐
         │ Dashboard (Visualization + API)          │
         │ - Pulls from Results DB                  │
         │ - Endpoint health trends & status        │
         └──────────────────────────────────────────┘
```
## Which DB to use?
### config DB
RDBMS:

I chose an RDBMS because we will later need ACID properties, meaning transactions. The database is sharded into an 
adequate number of shards to distribute the load and data. We use Active-Passive/Master-Slave Replication for each 
Partition in a semi-synchronous fashion. One Slave/Follower will follow synchronously while the others will receive 
the Replication Stream asynchronously. That way, 
we can be sure that at least one Slave holds up to date data in case the Master fails (due to network partitions, server outage, etc.) 
and that Slave will be promoted to be the new Leader.

### Results DB
Cassandra has a great write-throughput due to the fact that it is Leaderless (no time lost for fail overs for example), 
replicates asynchronously (can be configured) etc. We are also okay with Eventual Consistency because it is not crucial if we see a result with some delay.

## How to ensure no duplicate job gets scheduled?

<details>
  
- Use Strongly Consistent Config DB (like CockroachDB / PostgreSQL)
- Each job should have metadata:

```sql
job_id: UUID
endpoint_url: TEXT
last_checked_at: TIMESTAMP
next_check_at: TIMESTAMP
status: PENDING | IN_PROGRESS | DONE
```

### Job Scheduler Logic:
Runs periodically (e.g., every minute).

Queries DB for:

```sql
SELECT * FROM health_jobs
WHERE next_check_at <= now()
AND status = 'PENDING'
FOR UPDATE SKIP LOCKED;
FOR UPDATE SKIP LOCKED ensures no other instance of the scheduler can select the same row.
```
Prevents double scheduling in HA setup.

- Marks job as IN_PROGRESS in DB. Publishes job to Kafka scheduled-checks topic.
- (Optionally) Writes a job_execution_id (UUID) to track the execution instance.
- Eventually marks it as DONE (or retries if failed).

</details>

## understanding how Kafka + Worker Pool works under the hood?

<details>

  - Kafka organizes messages into Topics
  - Each topic is divided into Partitions (for parallelism)
  - Kafka ensures message ordering per partition
  - A pool of stateless services (usually Pods in K8s)
  - Each pod runs a Kafka consumer instance
  - All workers are part of the same Consumer Group

### Step 1: Topic and Partitioning
Say we have a Kafka topic:

```makefile
Topic: endpoint-health-check
Partitions: 5
```

Kafka distributes messages to these partitions — the partition assignment is usually based on a key, like endpoint_id, ensuring the same endpoint always goes to the same partition.

### Step 2: Workers Join Consumer Group
Let’s say we have 3 workers in our pool:

```yaml
Consumer Group: health-checker-group
  ├── Worker-1
  ├── Worker-2
  └── Worker-3
```
When they subscribe to endpoint-health-check topic, Kafka does the following:

### Partition Assignment
Kafka uses the Consumer Group Protocol to assign partitions:

```bash
Worker-1 gets: Partition 0, 1

Worker-2 gets: Partition 2, 3

Worker-3 gets: Partition 4
```
If a new worker joins (e.g., autoscaled by KEDA), Kafka rebalance happens, and the partitions are redistributed evenly.

### Let’s break it down with a concrete example:
📌 Scenario:
You want to monitor 6 endpoints:

api.service1.com/health

api.service2.com/health

...

api.service6.com/health

You have:

Kafka Topic: scheduled-checks

Partitions: 3

Worker Pool: 3 Pods (each a Kafka consumer in the same consumer group)

![image](https://github.com/user-attachments/assets/87eff32d-25d2-4b6f-83e2-0c4ddc6c1d8d)

So service1 and service4 go to Worker-1.

service2 and service5 go to Worker-2, and so on.

### who decides partition count?
You, the system designer or infrastructure/devops team, decide the number of partitions when creating a Kafka topic.

</details>

## if we integrate KEDA how autoscaling will work , how new workers will be assigned to same consumer group?

<details>
  
  - worker pool is a Kubernetes Deployment running multiple replicas (Pods).
  - Each Pod runs a Kafka consumer configured with the same consumer group ID.
  - Kafka automatically assigns partitions among all consumer instances (Pods) in that group — so work is divided without overlap.
  - For autoscaling, you deploy KEDA, which watches Kafka lag metrics.
  - KEDA dynamically scales the Deployment replicas up/down based on lag thresholds.
  - As pods scale, new consumers join the same consumer group, triggering Kafka’s partition rebalancing to distribute load evenly.
  - The minReplicaCount and maxReplicaCount for autoscaling with KEDA are defined in the KEDA ScaledObject resource YAML.

Here’s how and where you specify them:

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: kafka-worker-scaledobject
  namespace: your-namespace
spec:
  scaleTargetRef:
    name: kafka-worker-deployment   # your Deployment name
  minReplicaCount: 2                # minimum number of pods to keep running
  maxReplicaCount: 20               # maximum number of pods allowed
  triggers:
  - type: kafka
    metadata:
      bootstrapServers: kafka-broker:9092
      topic: endpoint-health-check
      consumerGroup: health-checker-group
      lagThreshold: "500"
```
- minReplicaCount: The smallest number of worker pods KEDA will keep running, even if lag is zero.
- maxReplicaCount: The maximum number of pods KEDA will scale up to, no matter how large the lag grows.


</details>


## Why to use API gateway? HA and scalability to handle in API gateway?
##  How to ensure HA in job scheduler service?
## job scheduler service Leader elected via K8s Lease / Redis / Zookeeper
https://dev.to/sklarsa/how-to-add-kubernetes-powered-leader-election-to-your-go-apps-57jh
https://faun.pub/understanding-kubernetes-leases-from-concept-to-implementation-bc423e868276

![image](https://github.com/user-attachments/assets/22d2429b-6050-4d32-9cda-83d8fb7f7c32)



