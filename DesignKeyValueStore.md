# Design a Key value Store

## Functional Requirements:
1. Set a key with a value (Update the value if it already exists).
2. Get the value given by the key.
3. Delete the key value (Unset the key).

## Non Functional Requirements:
1. Highly Scalable (Should be flexible to increase instances at runtime during load increase)
2. Consistent (Should provide consistent and correct response on each call).
3. Durable (No data should be lost during network partition failures).
4. Availability (No single point of failure). From CAP theorem, It’s a CP, means Consistency is more on priority than availability.

## Followup Questions:

👉What is the amount of data that we need to store?
Answer: Let’s assume a few 100 TB.

👉Do we need to support updates?
A: Yes.

👉Can a value be so big that it does not fit on a single machine?

A: No. Let’s assume that there is an upper cap of 1GB to the size of the value.

👉What would the estimated queries per second (QPS) be for this DB?

A: Let’s assume around 100k.

👉What is the minimum number of machines required to store the data?

A: As data to be stored is 100 TB, and each record can be as large as 1 GB. Let’s assume a storage capacity of each of the storage machine as 2 TB. Thus the minimum number of machines required will be 50.

👉Consistency vs Availability?

we would need to compromise with availability if we have tight consistency and partitioning. As is the case with any storage system, data loss is not acceptable.

👉Is sharding required?

A: Let’s look at our earlier estimate about the data to be stored. 100TB of data can’t be stored on a single machine.
Let’s say that we somehow have a really beefy machine which can store that amount of data, that machine would have to handle all of the queries ( All of the load ) which could lead to a significant performance hit.



