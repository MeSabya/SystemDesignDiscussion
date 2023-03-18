# Designing a URL Shortening service like TinyURL

## Why do we need URL shortening?

For example, if we shorten the following URL through TinyURL:

https://www.educative.io/courses/grokking-the-system-design-interview/m2ygV4E81AR

We would get:

https://tinyurl.com/rxcsyr3r

The shortened URL is nearly one-third the size of the actual URL

## Requirements and Goals of the System

### Functional Requirements:

- Given a URL, our service should generate a shorter and unique alias of it. This is called a short link. This link should be short enough to be easily copied and pasted into applications.
- When users access a short link, our service should redirect them to the original link.
- Users should optionally be able to pick a custom short link for their URL.
- Links will expire after a standard default timespan. Users should be able to specify the expiration time.

### Non-Functional Requirements:

- The system should be highly available. This is required because, if our service is down, all the URL redirections will start failing. (AP System)
- URL redirection should happen in real-time with minimal latency.
- Shortened links should not be guessable (not predictable).
- Analytics; e.g., how many times a redirection happened?
- Our service should also be accessible through REST APIs by other services.

## Capacity Estimation and Constraints
👉 Our system will be read-heavy. There will be lots of redirection requests compared to new URL shortenings. Let’s assume a 100:1 ratio between read and write.
### Traffic estimates
Assuming, we will have 500M new URL shortenings per month, with 100:1 read/write ratio, we can expect 50B redirections during the same period:

100 * 500M => 50B
What would be Queries Per Second (QPS) for our system? New URLs shortenings per second:

500 million / (30 days * 24 hours * 3600 seconds) = ~200 URLs/s
Considering 100:1 read/write ratio, URLs redirections per second will be:

100 * 200 URLs/s = 20K/s

### Storage estimates
Let’s assume we store every URL shortening request (and associated shortened link) for 5 years. Since we expect to have 500M new URLs every month, the total number of objects we expect to store will be 30 billion:

500 million * 5 years * 12 months = 30 billion
Let’s assume that each stored object will be approximately 500 bytes (just a ballpark estimate–we will dig into it later). We will need 15TB of total storage:

30 billion * 500 bytes = 15 TB
In the following table, we can change our assumptions to see how the estimates change:

![image](https://user-images.githubusercontent.com/33947539/154063117-7431bf5c-ef56-4836-98c5-b69361ff592f.png)

### Bandwidth estimates: 

For write requests, since we expect 200 new URLs every second, total incoming data for our service will be 100KB per second:

200 * 500 bytes = 100 KB/s
For read requests, since every second we expect ~20K URLs redirections, total outgoing data for our service would be 10MB per second:

20K * 500 bytes = ~10 MB/s

### Memory estimates: 

If we want to cache some of the hot URLs that are frequently accessed, how much memory will we need to store them? If we follow the 80-20 rule, meaning 20% of URLs generate 80% of traffic, we would like to cache these 20% hot URLs.

Since we have 20K requests per second, we will be getting 1.7 billion requests per day:

20K * 3600 seconds * 24 hours = ~1.7 billion
To cache 20% of these requests, we will need 170GB of memory.

0.2 * 1.7 billion * 500 bytes = ~170GB
One thing to note here is that since there will be many duplicate requests (of the same URL), our actual memory usage will be less than 170GB.

## System APIs
We can have SOAP or REST APIs to expose the functionality of our service. Following could be the definitions of the APIs for creating and deleting URLs:

          createURL(api_dev_key, original_url, custom_alias=None, user_name=None, expire_date=None)
          deleteURL(api_dev_key, url_key)

👉 **How do we detect and prevent abuse?** 
A malicious user can put us out of business by consuming all URL keys in the current design. To prevent abuse, we can limit users via their api_dev_key. Each api_dev_key can be limited to a certain number of URL creations and redirections per some time period (which may be set to a different duration per developer key).

## Database Design

- We need to store billions of records.
- Each object we store is small (less than 1K).
- There are no relationships between records—other than storing which user created a URL.
- Our service is read-heavy.
- Our system is AP.

### Database Schema:

![image](https://user-images.githubusercontent.com/33947539/154063705-a11d2411-d721-4175-b7c8-3791dbe4020c.png)

```Lua
What kind of database should we use? Since we anticipate storing billions of rows, and
we don’t need to use relationships between objects – a NoSQL store like DynamoDB, 
Cassandra or Riak is a better choice.
```

## Basic System Design and Algorithm

The problem we are solving here is how to generate a short and unique key for a given URL.

In the TinyURL example in Section 1, the shortened URL is “https://tinyurl.com/rxcsyr3r”. The last eight characters of this URL constitute the short key we want to generate. We’ll explore two solutions here:

### Solution1: Encoding actual URL

*We can compute a unique hash (e.g., MD5 or SHA256, etc.) of the given URL. The hash can then be encoded using base32 or base64.*
The reasonable question would be, what should be the length of short key ? 6,8 or 10.

Using base64 encoding, a 6 letters long key would result in 64^6 = ~68.7 billion possible strings.
Using base64 encoding, an 8 letters long key would result in 64^8 = ~281 trillion possible strings.

With 68.7B unique strings, let’s assume six letter keys would suffice for our system.

👉 ***Couple of problems with our approach ***:

If we use the MD5 algorithm as our hash function, it will produce a 128-bit hash value. 
After base64 encoding, we’ll get a string having more than 21 characters (since each base64 character encodes 6 bits of the hash value).

we only have space for 6 (or 8) characters per short key; **how will we choose our key then?**

We can take the first 6 (or 8) letters for the key. This could result in key duplication; to resolve that, we can choose some other characters out of the encoding string or swap some characters.

**What are the different issues with our solution?** 

We have the following couple of problems with our encoding scheme:

If multiple users enter the same URL, they can get the same shortened URL, which is not acceptable.
What if parts of the URL are URL-encoded? e.g., http://www.educative.io/distributed.php?id=design, and http://www.educative.io/distributed.php%3Fid%3Ddesign are identical except for the URL encoding.

**Workaround for the issues**:

👉 We can append an increasing sequence number to each input URL to make it unique and then generate its hash. But It may cause overflow.

👉 Another solution could be to append the user id (which should be unique) to the input URL. However, if the user has not signed in, we would have to ask the user to choose a uniqueness key. Even after this, if we have a conflict, we have to keep generating a key until we get a unique one.

### Solution2: Generating keys offline

*We can have a standalone Key Generation Service (KGS) that generates random six-letter strings beforehand and stores them in a database (let’s call it key-DB). Whenever we want to shorten a URL, we will take one of the already-generated keys and use it. This approach will make things quite simple and fast. Not only are we not encoding the URL, but we won’t have to worry about duplications or collisions. KGS will make sure all the keys inserted into key-DB are unique*


**What would be the key-DB size?**

With base64 encoding, we can generate 68.7B unique six letters keys. If we need one byte to store one alpha-numeric character, we can store all these keys in:

6 (characters per key) * 68.7B (unique keys) = 412 GB.

**Isn’t KGS a single point of failure?**

Yes, it is. To solve this, we can have a standby replica of KGS. Whenever the primary server dies, the standby server can take over to generate and provide keys.

![image](https://user-images.githubusercontent.com/33947539/154065945-774dceba-c122-460a-937f-fc5285414dff.png)

## Data Partitioning and Replication

👉 To scale out our DB, we need to partition it so that it can store information about billions of URLs. Therefore, we need to develop a partitioning scheme that would divide and store our data into different DB servers.

### Range Based Partitioning: 

We can store URLs in separate partitions based on the hash key’s first letter. Hence we will save all the URL hash keys starting with the letter ‘A’ (and ‘a’) in one partition, save those that start with the letter ‘B’ in another partition, and so on. This approach is called range-based partitioning.

The main problem with this approach is that it can lead to unbalanced DB servers. For example, we decide to put all URLs starting with the letter ‘E’ into a DB partition, but later we realize that we have too many URLs that start with the letter ‘E.’

### Hash-Based Partitioning:

In this scheme, we take a hash of the object we are storing. We then calculate which partition to use based upon the hash. In our case, we can take the hash of the ‘key’ or the short link to determine the partition in which we store the data object.

This approach can still lead to overloaded partitions, which can be solved using **Consistent Hashing.**

## Cache

We can cache URLs that are frequently accessed. We can use any off-the-shelf solution like Memcached, which can store full URLs with their respective hashes. Thus, the application servers, before hitting the backend storage, can quickly check if the cache has the desired URL.

### How much cache memory should we have?
We can start with 20% of daily traffic and, based on clients’ usage patterns, we can adjust how many cache servers we need. As estimated above, we need 170GB of memory to cache 20% of daily traffic. Since a modern-day server can have 256GB of memory, we can easily fit all the cache into one machine. Alternatively, we can use a couple of smaller servers to store all these hot URLs.

### Which cache eviction policy would best fit our needs?

When the cache is full, and we want to replace a link with a newer/hotter URL, how would we choose? Least Recently Used (LRU) can be a reasonable policy for our system.


## Load Balancer (LB)
We can add a Load balancing layer at three places in our system:

        Between Clients and Application servers
        Between Application Servers and database servers
        Between Application Servers and Cache servers

**Which Load balancer algorithm is to use ?**

Initially, we could use a simple Round Robin approach that distributes incoming requests equally among backend servers. This LB is simple to implement and does not introduce any overhead. Another benefit of this approach is that if a server is dead, LB will take it out of the rotation and stop sending any traffic to it.

A problem with Round Robin LB is that we do not consider the server load. As a result, if a server is overloaded or slow, the LB will not stop sending new requests to that server. To handle this, a more intelligent LB solution can be placed that periodically queries the backend server about its load and adjusts traffic based on that.

![image](https://user-images.githubusercontent.com/33947539/154067001-cf06bc94-f739-43b4-814f-1c168bda87a1.png)

## When to clean up the DB entries?

- A separate Cleanup service can run periodically to remove expired links from our storage and cache. This service should be very lightweight and scheduled to run only when the user traffic is expected to be low.
- We can have a default expiration time for each link (e.g., two years).
- After removing an expired link, we can put the key back in the key-DB to be reused.
- Should we remove links that haven’t been visited in some length of time, say six months? This could be tricky. Since storage is getting cheap, we can decide to keep links forever.





