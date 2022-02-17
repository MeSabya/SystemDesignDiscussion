# Distributed cache system design

## What is Cache?
Cache is a small memory, fast access local store where we store frequently accessed data. 
Caching is the technique of storing copies of frequently used application data in a layer of smaller, faster memory in order to improve data retrieval times, 
throughput, and compute costs.

## Why do we use Cache?

Cache is based on the principle of locality. It means that frequently accessed data is kept close to the system. 

👉 The two kinds of locality are:

**Temporal locality**: where data that has been referenced recently is likely to be referenced again (i.e. time-based locality).
It repeatedly refers to same data in short time span.

**Spatial locality**: where data that is stored near recently referenced data is also likely to be referenced again(i.e. space-based locality)
It only refers to data item which are closed together in memory. eg. Data stored together in an array

## How does a Cache work?

When a request comes to a system, there can be two scenarios. 

1. If a copy of the data exists in cache it’s called a **cache hit**, and when the data has to be fetched from the primary data store it’s called a **cache miss**. 
   The performance of a cache is measured by the number of cache hits out of the total number of requests.

## Types of Cache
**Application server cache**: Placing a cache directly on an application server enables the local storage of response data. Each time a request is made to the service, the server will return local cached data if it exists. If it is not in the cache, the requesting node will query the data from primary store. The cache could be located both in memory (which is very fast) and on the node’s local disk (faster than going to network storage).
The problem with this type of caching comes in when you have a distributed system. It leads to a lot of cache misses because each instance has its own cache which does not contain information on other instances.

**Distributed cache**: Each node will have a part of the whole cache space, and then using the consistent hashing function each request can be routed to where the cache request could be found.
The cache is divided up using consistent hashing and each request can be routed to the node which contains the response for it.

**Global Cache**: A global cache is a single cache memory in front of the server instances. It is common for all instances and fetches data from the primary store in case it’s not present in the cache itself. It has added latency but has the advantage of considerably better cache performance.

**CDN: CDN(Content Distribution Network)**
is used where a large amount of static content is served. The response can be HTML file, CSS file, JavaScript file, pictures, videos, etc. First, request ask the CDN for data, if it exists then the data will be returned. If not, the CDN will query the backend servers and then cache it locally.

## Cache Writing Policies or Cache Invalidation

A Cache Policy is a set of rules which define how the data will be loaded (and evicted) from a cache memory. A cache is made of copies of data, and is thus transient storage, so when writing we need to decide when to write to the cache and when to write to the primary data store.

If the data is modified in the database, it should be invalidated in the cache, if not, this can cause inconsistent application behavior

The most common cache writing policies are as follows:

**Write-through caching**: 

Under this scheme, data is written into the cache and the corresponding database at the same time. The cache ensures fast retrieval, and since data is written simultaneously in the cache and primary data store, there is complete consistency.
However, since every write operation has to be done twice it accounts for more latency.

![image](https://user-images.githubusercontent.com/33947539/148814313-ecf737d4-8898-4599-a4ac-f997a2d9bce8.png)

**Write-around caching**: 

In write around caching, data is written directly to primary data store. The cache in turn checks with primary data store to keep itself in sync. There is a downside here that cache might be behind the primary storage at times, but the added latency is countered by the primary data store always being consistent.

![image](https://user-images.githubusercontent.com/33947539/148814860-fe7376f6-8b5f-47f4-9708-63eefe51d22f.png)

**Write-back caching**: 

In write back caching data is written first into the cache, and then into the primary data store. There can be two scenarios here, either the primary storage is updated directly after the cache, or that the cache memory is persisted after an entry removal (in case the cache memory was already full). In this scenario the entry is tagged with a dirty bit to mark that the data is out of sync.
Write back caching is prone to data loss and should only be used in write-heavy operations where write speed is very important.

![image](https://user-images.githubusercontent.com/33947539/148815259-acc9be5b-be3e-4739-80fb-5dae132f8cfd.png)


## Cache Eviction Policies

*Cache eviction policies define a set of rules that decide what data must be removed when the cache is full and a new entry is to be added.*

A good replacement policy will ensure that the cached data is as relevant as possible to the application, that is, it utilises the principle of locality to optimise for cache hits.
Therefore a cache eviction policy can only be defined on the basis of what data is stored in the cache.

Some of the popular cache eviction algorithms are:

***First In First Out (FIFO)***: This cache evicts the first entry in the cache regardless of how many times it was called.

***Least Recently Used (LRU)***: Evicts the least recently used items first.

***Most Recently Used (MRU)***: Evicts the most recently used items first.

***Least Frequently Used (LFU)***: Counts how often an entry was read from cache. Those that are used least often are discarded first.

***Random Replacement (RR)***: Randomly selects a candidate item and discards it to make space when necessary.

## Distributed caches
The cache is divided up using a consistent hashing function and each of its nodes owns part of the cached data. If a requesting node is looking for a certain piece of data, it can quickly use the hashing function to locate the information within the distributed cache to determine if the data is available.

Pros: Cache space can be increased easily by adding more nodes to the request pool.
Cons: A missing node can lead to cache loss. We may get around this issue by storing multiple copies of the data on different nodes.

## Design a Cache System:

A single machine is going to handle 1M QPS
Map and LinkedList should be used as the data structures. We may get better performance on the double-pointer linked-list on the remove operation.
Master-slave technique

