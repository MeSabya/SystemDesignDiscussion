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

In our use case, complex relational queries are not used. Most data access can be described as primary-key queries (e.g. given a job ID, get all executions).

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

### Deep Dive
**API Server**:

A single machine is not a good idea while designing a distributed system, as it becomes the single point of failure. We need multiple replicas of the API server; in front of all the API servers, we need a load balancer. The load balancer can use a round-robin algorithm to distribute the request.

**Task Scheduler**:

The previous design merged both task runner and scheduler into the same component. We should split that component so that each element can have a single responsibility. Also, it will help to scale each service/component individually. We can break that component into two service/component

      - Task scheduler
      - Task runner

The task scheduler will run a query into the database to get the jobs due at a specific minute. Then all the due jobs or tasks will be enqueued to a distributed message queue such as SQS or RabbitMQ. First-in-first-out (FIFO) queue would be the best.
We should have a primary-secondary configuration for the task scheduler to remove the single point of failure. If the primary server fails, secondary will take over.

**Task Runner**:

Task runner service will fetch messages from the message queue. Each message will contain the job id and the URL to hit. Task runner can hit the URL then update the job status on the database. Task runner will also add the job back to the database if it is recurrent. Another service can do this rescheduling part if we want to split the service a little bit more.
We should have multiple task runners to handle 1000 tasks per second. If one server can handle four tasks per second, 250 task runners are needed to reduce the operation time between the scheduled and task start times.



