## What is a distributed lock 

A distributed lock is a synchronization mechanism that ensures that only one instance of a process or service can perform a specific operation at a given time 
across multiple nodes in a distributed system. It prevents race conditions and ensures data consistency in environments where multiple processes or servers operate concurrently.

## Scenario: Preventing Double Booking in an E-Commerce System
Imagine you are building an e-commerce platform where multiple users can place orders for the same product. There’s only 1 item left in stock, and two users try to purchase it at the exact same time from different servers.

### Problem Without a Distributed Lock
- User A and User B both see that 1 item is available.
- They both attempt to place an order simultaneously.
- Each server checks stock and sees 1 available item.
- Both transactions are processed, and both users are charged.
- Now, the system has oversold the item, causing order inconsistencies and customer complaints.

👉 If you have a single database instance, a simple DB lock (SELECT ... FOR UPDATE) might be enough.
👉 If you have multiple application servers or distributed databases, a distributed lock is necessary to prevent race conditions.

## More explanation.. Why a Distributed Lock is Needed in a Multi-Server or Distributed Database Setup

In a distributed system (multiple application servers and/or multiple database instances), ensuring mutual exclusion for critical operations becomes challenging because:

- Different servers can query different database replicas
- Replication lag can cause stale reads
- Database locks are local to a single instance and do not synchronize across multiple nodes

A distributed lock ensures that only one process can execute a critical operation at any given time across all nodes.

### Scenario: Online Ticket Booking with Distributed Databases

Setup:
- Multiple Application Servers (e.g., App Server 1 and App Server 2)
- Multiple Database Instances (e.g., DB Replica 1 and DB Replica 2)
- Users are booking tickets for a concert where only 1 seat is left

### Problem Without a Distributed Lock
👉 Step 1:

User A’s request goes to App Server 1, which queries DB Replica 1 and sees 1 seat available.
User B’s request goes to App Server 2, which queries DB Replica 2 and also sees 1 seat available.

👉 Step 2:

Both servers process the requests at the same time because their respective DB replicas still show 1 seat available due to replication lag.
Both users are confirmed for the same seat (overselling occurs).

👉 Step 3:

The databases eventually synchronize, but two tickets have been issued for one seat, leading to an inconsistent state.

### Why a Simple DB Lock Won't Work?
1. Database locks are local to a single instance

SELECT ... FOR UPDATE works only on the specific database node it was executed on.
If two app servers talk to different DB replicas, they won't see each other’s locks.

2. Replication Lag Issue

Even if one instance locks the seat, another might read stale data from a lagging replica, leading to race conditions.

3. Scalability Issues

Keeping all transactions locked in a central database leads to performance bottlenecks.
A high-traffic system cannot rely solely on database locking for consistency.

### three common High Availability (HA) solutions for Redis distributed locks

There are three common High Availability (HA) solutions for Redis distributed locks, but not all are used in real-world production environments due to trade-offs. Let’s break them down and see which ones are actually used in practice.

1️⃣ Redlock (Multi-Redis Nodes) ✅ [Common in Realtime]
How It Works:
Uses multiple independent Redis nodes (e.g., 5 instances).
A lock is only valid if acquired on a majority (N/2 + 1) of nodes.
Avoids split-brain by requiring quorum-based consensus.
Locks expire automatically to prevent deadlocks.
✅ Used in real-time:

Works well for distributed microservices and rate limiters.
Ensures consistency across multiple Redis instances.
Redisson, Spring Boot, and AWS ElastiCache support Redlock.
⚠️ Limitations:

Requires at least 3-5 Redis instances, increasing operational complexity.
Small chance of race conditions due to network latency.
2️⃣ Redis Sentinel (Leader-Follower Failover) ❌ [Not Used for Locks]
How It Works:
Sentinel monitors Redis master and automatically promotes a new master if the current one fails.
Applications always write to the master, and Sentinel updates clients if the master changes.
❌ Not used for locks:

Sentinel-based failover is slow (failover takes seconds).
During failover, the new master doesn’t inherit the old locks, leading to potential data inconsistency.
✅ Used for:

Redis caching (to keep data available if the master fails).
Not recommended for distributed locking.
3️⃣ Redis Cluster (Sharded Redis) ⚠️ [Rarely Used for Locks]
How It Works:
Redis Cluster shards data across multiple Redis nodes.
Each key is stored in a single shard, meaning a lock only exists on one shard.
Provides high availability with automatic failover.
⚠️ Rarely used for locks:

If a node fails, the lock is lost unless it’s replicated elsewhere.
No quorum-based locking, so race conditions can still occur.
Sharding makes lock coordination harder because different keys go to different shards.
✅ Used for:

Large-scale caching, session storage, and message queues.
Not ideal for distributed locks.

