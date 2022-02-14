# Distributed Job Scheduler 

where you define a job to be scheduled at a specific time and this job can be one-off or recurring.

## Functional requirements
- It will be a service, and customers can schedule a job on demand.
- Customers will provide a cron definition and a task for scheduling a job
- Initially, the task is simple, and customers will provide a URL, the system needs to hit that URL during the scheduled time. 
- The same task can be repeated.
- Customers can define a job as low as 1 minute and as high as 30 days.

## Non-functional requirements
- Scalability: Thousands or even millions of jobs can be scheduled and run per day.
- Durability: Jobs must not get lost -> we need to persist jobs.
- Reliability: Jobs must not be executed much later than expected or dropped -> we need a fault-tolerant system
- Availability: It should always be possible to schedule and execute jobs -> (dynamical) horizontal scaling
- Jobs must not be executed multiple times (or such occurences should be kept to a minimum)

👉 **Based on the estimations above, we can make the following conclusions:**

- The backend database should be horizontally scalable, as the execution history grows quickly (~ 100 million rows of data created every day).
- The system is read-heavy, as a job is created once and read many times by the scheduler. The same holds true for execution history, an entry is created and can be queried many times by the user.
- We need a distributed group of workers to run jobs concurrently with varying capabilities.
- All critical services should be replicated to handle the large traffic.

## Database Design:
In database design, the access pattern of your application determines what schema is used. Let’s investigate the requests that hit the database:

**Read Operations**:

- Given a user ID, retrieve all jobs that belong to it (by client)
- Given a job ID, retrieve all/latest execution histories belonging to it (by client)
- Find all jobs that are scheduled to run right now (by internal servers)

**Write Operations**:

- A user can create/delete a new job schedule (by client).
- The workers will add execution histories to the database (by internal servers).
- The system updates the next execution timestamp of a job after running it (by internal servers).

In our use case, complex relational queries are not used. Most data access can be described as primary-key queries (e.g. given a job ID, get all executions). We need strong consistency. So we can use shraded SQL or NO SQL DB like HBase or Postgres.

#### Schema

**Job Table**: We need a table that keeps track of job metadata such as owners, execution intervals, and retry policies. Remember the access pattern for this table: given a user ID, get some job records. Hence, we can use UserID as the partition key, and JobID as the sort key.

![image](https://user-images.githubusercontent.com/33947539/153820350-338e56fd-9ee9-4c8a-9b91-696d28fb4f6f.png)

**History Table**: This table is used to store execution details of a job. Given a job, there could be multiple executions associated with it. Remember the access pattern: given a JobID, retrieve all execution histories. Hence, we use JobID as the partition key and execution ID as the sort key (the execution ID can be a simple timestamp)

![image](https://user-images.githubusercontent.com/33947539/153821701-a54fba24-9b3e-4eb8-b88d-8e1ed13422b3.png)

**Schedule Table**: The core feature of any job scheduler is, of course, running jobs on time. Therefore, we need a data model that makes it easy to filter jobs by execution time. Below is a simple design that works well for a small amount of data:

![image](https://user-images.githubusercontent.com/33947539/153821765-f5e065e4-7ff6-421f-89f0-68d1abed8fb5.png)

### API Design

Our application is easy to use, it only needs the following RPC interfaces:

            submit_job(user_id, schedule, code_location)
            retrieve_all_jobs(user_id)
            delete_job(user_id, job_id)
            get_exec_history(job_id)

## Initial design thoughts

It's always a good idea to go from a single machine or most straightforward design first, then try to scale or optimize the solution. Here is my initial idea

![image](https://user-images.githubusercontent.com/33947539/153815620-c4183956-d429-4cbc-8ee3-4b8965467841.png)

Here we have three components:

**API server**: A HTTP web server where customers can hit directly to add jobs.

**Bigtable/HBase**: This is the database where we will store the job-related information. Why Bigtable or Hbase? It's because we need strong consistency for our system, and our write volume will be pretty high for which Bigtable/Hbase works pretty well than other solutions. It seems we don't require ACID property, so Bigtable/Hbase fits our requirement pretty well, also highly scalable. According to CAP theorem, Bigtable/Hbase is CP (Consistency, Partition tolerance) system, so it feels like it will lose availability. Still, in real life, it has an availability of 99.9 percentile, so it's a highly available system as well.

**Task Runner**: It is a daemon that will run in the background. Every minute it will run a query on the DB to find out the due jobs and run them concurrently as much as possible.

### Issues with this design
*There are multiple single points of failure. The most crucial component is the task runner which needs to find out the due jobs and run the tasks. One single machine will not be able to run 1000 tasks concurrently. We need more machines to handle this task run load. Another issue is we need to run a query on the DB table to get the due jobs every single minute. We definitely need indexing to reduce the query load. So can we make the design better?*

![image](https://user-images.githubusercontent.com/33947539/153816332-4b60f5c5-9302-409f-a1f6-427f1af820dc.png)

### Architecture Deep Dive

**Web Service**: The gateway to the scheduling system. All RPC calls from the client are handled by one of the RPC servers in this service.

**Scheduling Service**: It checks the database every minute for pending jobs and pushes them to a queue for execution. Once a job is scheduled, create an entry in the execution history table with status = SCHEDULED. With this service, we guarantee that all jobs are pushed to the queue in a timely manner.

**Execution Service**: In this service, we manage a large group of execution workers. Each worker is a consumer and executes whatever jobs it gets from the queue. Additional bookkeeping is needed to ensure re-execution upon worker failures.


![image](https://user-images.githubusercontent.com/33947539/153833275-fa34e4b9-afbb-42a7-a918-2d5d9822aa1e.png)

#### Run tasks on schedule

Let’s examine the query statement again:
            SELECT * FROM ScheduleTable WHERE NextExecution > "1641082500" AND NextExecution < "1641082580"

When the number of jobs running concurrently is small, the query yields a reasonable amount of data. However, if, let’s say, 100K jobs are scheduled to run in this minute, we certainly need more workers to handle the ingress data from the query as well as push messages to the queue.
The complexity arises when multiple workers are introduced — How can we distribute the data so that each worker only consumes a small fraction (10%) of the 100K jobs? It turns out that a simple composite partition key can fix this issue:

![image](https://user-images.githubusercontent.com/33947539/153838205-43e9e3ba-93f1-4471-9eb6-d1eff31a33c7.png)

When a row is added to the schedule table, it is randomly assigned a shard number. With the new schema, it is super easy to distribute the load across workers:

```SQL
Worker 1: 
SELECT * FROM ScheduleTable WHERE NextExecution > "1641082500" AND NextExecution < "1641082580" AND shard =1 
Worker 2:
SELECT * FROM ScheduleTable WHERE NextExecution > "1641082500" AND NextExecution < "1641082580" AND shard = 2
```

👉 **The problem becomes how can we assign N shards evenly to M workers, where M can change at any time?**

![image](https://user-images.githubusercontent.com/33947539/153838454-dab8d935-988a-4206-ae2b-b2dec3482b2e.png)

To make sure a complete work assignment, we can borrow some ideas from MapReduce, where a master is used to assign and monitor workers. If a worker dies, the master will resend its work to some other nodes. An additional local database is used so that no job is scheduled twice. When a job is pushed to the queue, an entry is created in the local DB with 2 minutes of expiration time. If the original handler of the record dies and the shard is handed over to another worker, the new worker will skip tasks that exist in the database.

#### Queue delivery

A queue is introduced for two reasons. Firstly, we want a buffer that holds all pending jobs at peak hours. Secondly, it decouples the two services.

#### Handling failures & Retries

Although at-least-once delivery guarantees that every job gets assigned to a worker, it does not prevent jobs from getting dropped upon worker failure. To achieve true retry capability, an asynchronous health checker is introduced to run the following logic:

1. When a job is assigned, an entry is created in the local database with the latest update time.
2. All workers are required to refresh the update time ~10s.
3. The health checker will periodically scan the table, looking for jobs with stale update time (e.g. older than 30 seconds).
4. Jobs that meet the above criteria are sent back to the queue for re-execution.

![image](https://user-images.githubusercontent.com/33947539/153839734-dd55a55c-0e83-4245-945c-25efbae6c659.png)

### Data flow

With all the wrinkles in the high-level design address, we can finally come up with the data flow of the system:

##### Create/delete job/Retrieve history
The client sends out an RPC call to Web Service.
One of the RPC servers queries the database using the provided partition key and return the result

##### Schedule a job

**Master**
Every minute, the master node creates an authoritative UNIX timestamp and assigns a shard ID (see details for more) to each worker along with the timestamp
Check worker health regularly. If it dies, reassign its work to others

**Worker**
The worker queries the database with the timestamp and shard ID.
For each row, send it to the queue if it has not been scheduled (see details for more)

##### Execute a job

**Orchestrator**
A group of orchestrators consumes messages from the queue
Given a message, find one worker with the least workload. Assign the job to the worker
Commit the index, repeat steps 1 to 3

**Worker**
The worker regularly update the local database with its timestamp

**Health Checker**
Scans the local database ~ 10 seconds
If any row hasn’t been updated in ~30 seconds, retry it by pushing the job ID to the queue

## High Level Design 

![image](https://user-images.githubusercontent.com/33947539/153842042-bfedb2b1-b334-4b87-9627-61517dcc0851.png)

**Microservices** that want to schedule a non-/recuring Job: Can send a Message (or produce in Kafka terminology) to the corresponding Kafka Queue (precisely a Topic).

**Job Scheduler Service**: Will consume the Messages (requesting a Job enqueueing). They will generate a unique Id using e.g. the Snowflake ID Generation concept. Based on that ID (e.g. by hashing it) they decide into which Database Partition the Job will go. They create a Job and Trigger record according to the Message in the corresponding Database Partition.

**RDBMS**: 

I chose an RDBMS because we will later need ACID properties, meaning transactions. The database is sharded into an adequate number of shards to distribute the load and data. We use Active-Passive/Master-Slave Replication for each Partition in a semi-synchronous fashion. One Slave/Follower will follow synchronously while the others will receive the Replication Stream asynchronously. That way, we can be sure that at least one Slave holds up to date data in case the Master fails (due to network partitions, server outage, etc.) and that Slave will be promoted to be the new Leader.


**Job Executor Service**:

1. On Startup it will fetch the Database Partitioning info from ZooKeeper as well as the Partition Assignment between other instances and the Database Partitions.
2. It will choose a Database Partition which has the least number of Executors assigned to balance out the number of Executors that execute Jobs for a all the different Database Partitions.
3. It will send/store the Partition Assignment to ZooKeeper.
4. It constantly sends Heartbeats to ZooKeeper
5. It pulls information from the Database Partition and fights with other Executor instances assigned to the same Database Partition for Jobs that are due to execute. The fighting works by using Row Locks. That is why we need transactional properties (hence an RDBS that supports that, not all really do!)
6. Before an Executor Node executes a Job, it will update the Job record in the DB: Flagging it to be “Running”, store the “StartTime” and which Executor node (=itself) is executing the Job, etc.
7. When an Executor Node fails then other Nodes (assigned the same Partition) can detect it using ZooKeeper (due to the Heartbeating). They can then find all the Jobs that the failed Node was executing (Flag=Running and LastExecutor=Failed Node) and can fight for those Jobs to execute them (we could make a Job configurable to retry or not in case of execution failure).
8. Finally after successfully/unsuccessfully executing a Job, we send/publish/produce a Message to another Kafka Queue.

Regarding Point 7 (to expand the horizon on possibilities): We could also use Broadcast Messaging or a Gossip Protocol to detect Node failures. I'm excited to hear your argumentation.


**Result Handler Service**: 

Will consume Messages and store the Execution Result in a NoSQL database like Cassandra – Cassandra has a great write-throughput due to the fact that it is Leaderless (no time lost for fail overs for example), replicates asynchronously (can be configured) etc. We are also okay with Eventual Consistency because it is not crucial if we see a result with some delay.

**Coordination Service, ZooKeeper**: 

Stores the above mentioned information. Regarding the Database Partitioning information: We can e.g. load that info into ZooKeeper from another Service/Configuration file.

**Message Queues, in general**: 

We use Message Queues (here Kafka which is more of an (append) log than a Queue in the conventional sense, you can find great docs on the official website) in order to:

- Be able to scale the consumer and producer nodes independently (in Kafka we have Topics which can be horizontally partitioned and scale by that)
- We decouple the consumer and producer from each other
- Lower latency for the producer (doesn't have to wait for a response)
- Durability and Reliability: When a Consumer Node crashes another Node can process the Message which otherwise would be lost (see Offset in Kafka). The Messages are persisted.
- We can throttle/limit the number of messages the consumers process (see Backpressure)
- Kafka offers message ordering


## References:
https://leetcode.com/discuss/general-discussion/1082786/System-Design%3A-Designing-a-distributed-Job-Scheduler-or-Many-interesting-concepts-to-learn

https://towardsdatascience.com/ace-the-system-design-interview-job-scheduling-system-b25693817950
