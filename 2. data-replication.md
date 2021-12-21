Copying the same data over multiple nodes in distributed systems is critical to keep the database up and keep on serving queries even during faults.
Following are other reason behind data replication in a distributed system :

1. **Higher Availability**: To ensure the availability of the distributed system( System keeps on working even if one or fewer nodes fail).
2. **Reduced Latency**: Replication assists in reducing the latency of data queries (By keeping data geographically closer to a user. For example, CDN(Content Delivery Networks).
    keeps a copy of replicated data closer to the user. Ever thought how Netflix streams videos with such short latencies!)
3. **Read Scalability**: Read queries can be served from replicated copies of the same data (this increase overall throughput of queries).
4. **Network Interruption**: System works even under network faults.

## How does replication work?
There are three ways through which leaders replicate data to its follower :
1. Statement-based replication
2. Write-Ahead log
3. Row-based replication

### Statement-based replication
In this strategy leader logs every write statement(INSERT/UPDATE/DELETE) that it executes and sends the same to its follower and then the follower executes it on its node.
#### Drawback :
Any non-deterministic function can result in different writes on follower and leader(Like RAND() or NOW() functions). 

### Write-Ahead log
Whenever a query comes to a system, even before executing that query, 
it is written in an append-only log file also known as Write-ahead log file. This log file can be used to replicate data to follower nodes.
PostgreSQL and Oracle employ this strategy of data replication.

### Row-based replication
A row-based replication involves a sequence of records describing writes to the level of row. For example, an insertion of row contains information 
about all new values of columns. A delete contains information to identify rows to be deleted and an update contains new values of all columns.

## Approaches to data replication
- Single leader
- Multi leader
- leaderless

### Single leader replication
- In single leader replication, the leader(master or primary) replicates data to all of its followers (slaves, read replicas, secondary )nodes. 
  This is the most commonly used mode of replication
- Whenever a new write comes to the master node, it keeps that write to its local storage and sends the same data to all its replicas as a change stream or replication log. 
  Each slave then updates its own local copy of data in the same order as it was processed on the leader node.
- PostgreSQL, MySQL, and Oracle Data Guard and NoSQL databases like MongoDB, RethinkDB, and Espresso use this mode of replication.
- Message brokers like Kafka and queues like RabbitMQ also employ single leader based replication

### Failures and Recovery in a single leader replication
- **Failure of follower and it’s recovery**
  Each follower keeps a copy of data on its local storage. If the follower node is crashed or restarted or isn’t able to sync with the leader 
  for a long duration due to network issues, the follower can easily catch up with the leader with the help of its log offset. From the last offset in its log, 
  it can request the leader to send all data that it missed to copy due to temporary failure.
  After applying those changes followers get in sync with the leader and can continue to take further data changes.
  
- **Failure of leader and its recovery**
  1. Failure of leader requires promoting one of the followers as a new leader, 
     other followers to change their leader to the newly appointed one and all clients to reset its configuration and send all writes to this new leader.
  2. Nodes pass messages among themselves at regular interval and if one node doesn’t respond with a given duration, 
     then it’s assumed to be dead (Node may be still working but due to some network issue, it becomes temporarily unavailable. 
     It is still assumed to be dead as no other system can know for sure the reasons behind such non-responsiveness)
  3.  if the older leader comes back, it is forced to become a follower.
  
  
  

