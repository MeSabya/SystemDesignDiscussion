# Cloud storage system design

Let's design a file hosting service like Dropbox or Google Drive. Cloud file storage enables users to store their data on remote servers. Usually, these servers are maintained by cloud storage providers and made available to users over a network (typically through the Internet). Users pay for their cloud data storage on a monthly basis.

Similar Services: OneDrive, Google Drive

## Requirements 

Here are the top-level requirements for our system:

1. Users should be able to upload and download their files/photos from any device.
2. Users should be able to share files or folders with other users.
3. Our service should support automatic synchronization between devices, i.e., after updating a file on one device, it should get synchronized on all devices.
4. The system should support storing large files up to a GB.
5. ACID-ity is required. Atomicity, Consistency, Isolation and Durability of all file operations should be guaranteed.
6. Our system should support offline editing. Users should be able to add/delete/modify files while offline, and as soon as they come online, all their changes should be synced to the remote servers and other online devices.
7. The system should support snapshotting of the data, so that users can go back to any version of the files.

### Design Considerations 
👉 We should expect huge read and write volumes. Read to write ratio is expected to be nearly the same
👉 *Internally, files can be stored in small parts or chunks (say 4MB); this can provide a lot of benefits i.e. all failed operations shall only be retried for smaller parts of a file. If a user fails to upload a file, then only the failing chunk will be retried.*

👉 Keeping a local copy of the metadata (file name, size, etc.) with the client can save us a lot of round trips to the server. So cache needs to be implementaed.

**While providing file upload and sync as service it is not a good idea to save the file as a whole. Why ??**
1. **Bandwidth and cloud space utilization**: Whenever we want to sync the same file in different clients or keep multiple version of the file to provide history of updates to the file. It isn’t good idea to always backup and transfer the whole file to and fro as it takes more space!!
2. **Latency or Concurrency utilization**: It takes more time to upload single file as a whole. Also we cant upload file concurrently using multi threads or multi processes.

So the better model is to break the files in to multiple chunks and then its easier to upload, save and keep multiple version of files by just saving the chunks which are updated upon file update.
For small changes, clients can intelligently upload the diffs instead of the whole chunk.

## Capacity Estimation and Constraints:
Let’s assume that we have 500M (500, 000, 000) total users, and 100M(100, 000, 000) daily active users (DAU).
Let’s assume that on average each user connects from three different devices.
On average if a user has 200 files/photos, we will have 100 billion total files.

500, 000, 000 * 200 = 100,000000000 (100 billion)

Let’s assume that average file size is 100KB, this would give us ten petabytes of total storage.
100B * 100KB => 10PB (10 to the power 15)
Let’s also assume that we will have one million active connections per minute

## High Level System Design 

👉 The user will specify a folder as the workspace on their device. Any file/photo/folder placed in this folder will be uploaded to the cloud, and whenever a file is modified or deleted, it will be reflected in the same way in the cloud storage. The user can specify similar workspaces on all their devices and any modification done on one device will be propagated to all other devices to have the same view of the workspace everywhere.

👉 At a high level, we need to store files and their metadata information like File Name, File Size, Directory, etc., and who this file is shared with. So, we need some servers that can help the clients to upload/download files to Cloud Storage and some servers that can facilitate updating metadata about files and users. We also need some mechanism to notify all clients whenever an update happens so they can synchronize their files.

1. **Block servers** will work with the clients to upload/download files from cloud storage and 
2. **Metadata servers** will keep metadata of files updated in a SQL or NoSQL database. 
3. **Synchronization servers** will handle the workflow of notifying all clients about different changes for synchronization.

![image](https://user-images.githubusercontent.com/33947539/151672812-9774a266-0c7a-4b49-aa2a-44676591de55.png)


![image](https://user-images.githubusercontent.com/33947539/151672876-765d3643-b5bb-4a49-a1ea-907bc6507efc.png)


## Component Design

### Client
Lets assume we have file upload client installed on computer/mobile.

1. The Client Application monitors the workspace folder on the user’s machine and syncs all files/folders in it with the remote Cloud Storage. 
2. The client application will work with the storage servers to upload, download, and modify actual files to backend Cloud Storage. 
3. The client also interacts with the remote Synchronization Service to handle any file metadata updates, e.g., change in the file name, size, modification date, etc.

we can divide our client into four parts:

✔ **Internal Metadata database**: will keep track of all the files, chunks, their versions, and their location in the file system.

✔ **Chunker**: will split the files into smaller pieces called chunks. It will also be responsible for reconstructing a file from its chunks. Our chunking algorithm will detect the parts of the files that have been modified by the user and only transfer those parts to the Cloud Storage; this will save us bandwidth and synchronization time.

✔ **Watcher**: will monitor the local workspace folders and notify the Indexer (discussed below) of any action performed by the users, e.g. when users create, delete, or update files or folders. Watcher also listens to any changes happening on other clients that are broadcasted by Synchronization service.

✔ **Indexer**: will process the events received from the Watcher and update the internal metadata database with information about the chunks of the modified files. Once the chunks are successfully submitted/downloaded to the Cloud Storage, the Indexer will communicate with the remote Synchronization Service to broadcast changes to other clients and update the remote metadata database.

![image](https://user-images.githubusercontent.com/33947539/151673152-6a89ea17-a78c-405d-857b-f1a97d47d271.png)

### Metadata Database

1. The Metadata Database is responsible for maintaining the versioning and metadata information about files/chunks, users, and workspaces. 
2. The Metadata Database can be a relational database such as MySQL, or a NoSQL database. we just need to make sure we meet the data consistency.

The metadata Database should be storing information about following objects:

Chunks
Files
User
Devices
Workspace (sync folders)

### Synchronization Service
The Synchronization Service is the component that processes file updates made by a client and applies these changes to other subscribed clients. It also synchronizes clients’ local databases with the information stored in the Metadata Database. This is the most important part of the system architecture due to its critical role in managing the metadata and synchronizing users’ files. Desktop clients communicate with the Synchronization Service to either obtain updates from the Cloud Storage, or send files and updates to the Cloud Storage and potentially other users.
If a client was offline for a period of time, it polls the system for new updates as soon as it goes online. When the Synchronization Service receives an update request, it checks with the Metadata Database for consistency and then proceeds with the update. Subsequently, a notification is sent to all subscribed users or devices to report the file update.

### Message Queuing Service
An important part of our reference architecture is a messaging middleware that should be able to handle a substantial amount of reads and writes. 

A scalable Message Queuing Service that supports asynchronous message-based communication between **clients and the Synchronization Service** instances best fits the requirements of our application. 

The Message Queuing Service supports asynchronous and loosely coupled message-based communication between distributed components of the system. 
The Message Queuing Service should be of high performance, highly scalable, and be able to persistently store any number of messages in a highly available and reliable queue. 
The Message Queuing Service also provides load balancing and elasticity for multiple instances of the Synchronization Service. 

![image](https://user-images.githubusercontent.com/33947539/151673355-d925cd88-aff8-403c-85e0-b0b4c525a08c.png)

Figure above  illustrates two types of queues that are used in our Message Queuing Service. 
- The Request Queue is a global queue that is shared among all clients. 
- Clients’ requests to update the Metadata Database through the Synchronization Service will be sent to the Request Queue. 
- The Response Queues that correspond to individual subscribed clients are responsible for delivering the update messages to each client. 
- Since a message will be deleted from the queue once received by a client, we need to create separate Response Queues for each client to be able to share an update message which should be sent to multiple subscribed clients

![image](https://user-images.githubusercontent.com/33947539/151673437-4eb27374-3f81-468f-bca9-2e85ba87dfe3.png)

### Metadata Partitioning
To scale out metadata DB, we need to partition it so that it can store information about millions of users and billions of files/chunks. We need to come up with a partitioning scheme that would divide and store our data in different DB servers.

1. **Vertical Partitioning**: We can partition our database in such a way that we store tables related to one particular feature on one server. For example, we can store all the user-related tables in one database and all files/chunks related tables in another database. Although this approach is straightforward to implement it has some issues:

    **Will we still have scale issues?** 
    - What if we have trillions of chunks to be stored and our database cannot support storing such a huge number of records? How would we further partition such tables?
    - Joining two tables in two separate databases can cause performance and consistency issues. How frequently do we have to join user and file tables?

2. **Range Based Partitioning**: What if we store files/chunks in separate partitions based on the first letter of the File Path? In that case, we save all the files starting with the letter ‘A’ in one partition and those that start with the letter ‘B’ into another partition and so on. This approach is called range-based partitioning. We can even combine certain less frequently occurring letters into one database partition. We should come up with this partitioning scheme statically so that we can always store/find a file in a predictable manner.

    The main problem with this approach is that it can lead to unbalanced servers. For example, if we decide to put all files starting with the letter ‘E’ into a DB partition, and later we realize that we have too many files that start with the letter ‘E’, to such an extent that we cannot fit them into one DB partition.

3. **Hash-Based Partitioning**: In this scheme we take a hash of the object we are storing and based on this hash we figure out the DB partition to which this object should go. In our case, we can take the hash of the ‘FileID’ of the File object we are storing to determine the partition the file will be stored. Our hashing function will randomly distribute objects into different partitions, e.g., our hashing function can always map any ID to a number between [1…256], and this number would be the partition we will store our object.

### Caching
Memcached can be used between Cient and Block storage. 
Which cache replacement policy would best fit our needs? When the cache is full, and we want to replace a chunk with a newer/hotter chunk, how would we choose? Least Recently Used (LRU) can be a reasonable policy for our system. Under this policy, we discard the least recently used chunk first. Similarly, we can have a cache for Metadata DB.

### Load Balancer (LB)
We can add the Load balancing layer at two places in our system: 1) Between Clients and Block servers and 2) Between Clients and Metadata servers. Initially, a simple Round Robin approach can be adopted that distributes incoming requests equally among backend servers. This LB is simple to implement and does not introduce any overhead. Another benefit of this approach is if a server is dead, LB will take it out of the rotation and will stop sending any traffic to it. A problem with Round Robin LB is, it won’t take server load into consideration. If a server is overloaded or slow, the LB will not stop sending new requests to that server. To handle this, a more intelligent LB solution can be placed that periodically queries backend servers about their load and adjusts traffic based on that.

### Security, Permissions and File Sharing

One of the primary concerns users will have while storing their files in the cloud is the privacy and security of their data, especially since in our system users can share their files with other users or even make them public to share them with everyone. To handle this, we will be storing the permissions of each file in our metadata DB to reflect what files are visible or modifiable by any user.
This approach can still lead to overloaded partitions, which can be solved by using Consistent Hashing.











