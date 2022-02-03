# System Design — Top K Trending Hashtags

>Design a system to compute the top K trending hashtags in the last 24 hours for Twitter/Instagram.

**Similar questions**:

      Show the top 10 listened songs in the last 7 days per user on a music streaming app.
      Show the top 10 most viewed/clicked news article on LinkedIn in the last 24 hours.
      Show the top 100 most searched results in the last 24 hours on Google.
      
## Functional Requirements
Users should be able to see the top K trending hashtags in the last 24 hours. Here K can be dynamically updated and/or the time interval can be updated from 24 hours to say 1 hour and so on.

## Non Functional Requirements

- System should be highly available and scalable.
- Need not be strongly consistent i.e. eventual consistency
- Results could be approximate but with a very less error rate.
- Results should be returned with a very low latency

### Traffic and Throughput Requirements

- 500 million daily active users.
- Assuming 80% of the users write a tweet or re-tweet, number of new tweets per day = 400 million.
- Assuming each tweet has approximately 5 hashtags, number of hashtags per day = 2 billion.
- Assuming that 25% of the hashtags are unique, number of unique hashtags per day = 500 million
- Write Throughput = 2 billion/86400 = 23K hashtags per second
- Assuming that each user refreshes the feed or opens the feed 5 times in a day, number of read requests = 500 million*5 = 2.5 billion
- Read Throughput = 2.5 billion/86400 = 29K reads per second

## Distributed Database/File Storage
In order to persist the data, store the hashtags in a database or a file storage.
We can use a NoSQL database such as Cassandra to store the hashtags along with the timestamp with a schema something like (event_id, hashtag, timestamp) with the event_id as the partition key and the timestamp as the range key so that the data is sorted by timestamp.

Instead of storing the timestamp separately, we can use SnowflakeID format for the event_id which is ordered by the timestamp.
Assuming that we are storing hashtags for 1 year, the total number of hashtags = 365*2 billion = 730 billion.
Assuming that each hashtag is approximately 7 characters longs i.e. 7 bytes=56 bits, snowflake event_id is 64 bits, thus total storage requirement is 730 billion*120 bits = 10TB.
This much data can be accommodated on a single Cassandra instance.

We can choose to go ahead with asynchronous replication for our cluster since our requirement was eventual consistency.

Instead of an expensive Cassandra instance, we can also choose to use a distributed file storage system such as S3 or HDFS.
In S3 we can create folders for data for each day for the last 1 year and inside each folder have multiple sub-folders corresponding to each hashtag and inside each hashtag folder we have multiple files corresponding to the hashtag ingested for each hour.

### Reference
https://mecha-mind.medium.com/system-design-top-k-trending-hashtags-4e12de5bb846

