# System Design — Sharding / Data Partitioning

- Sharding is a method of splitting and storing a single logical dataset in multiple databases. 
- By distributing the data among multiple machines, a cluster of database systems can store larger dataset and handle additional requests. 
- Sharding is necessary if a dataset is too large to be stored in a single database. Moreover, many sharding strategies allow additional machines to be added. 
- Sharding allows a database cluster to scale along with its data and traffic growth.

>Data partitioning: It is the process of distributing data across a set of servers. It improves the scalability and performance of the system.


## Partitioning method
1. Horizontal partitioning — also known as sharding
2. Vertical partitioning
3. Directory-based partitioning

### Horizontal partitioning — also known as sharding
- It is a range-based sharding. You put different rows into different tables, 
  the structure of the original table stays the same in the new tables, i.e., we have the same number of columns.
- When partitioning your data, you need to assess the number of rows in the new tables, 
  so each table has the same number of data and will grow by a similar number of new customers in the future.

### Vertical partitioning
- This type of partition divides the table vertically (by columns), which means that the structure of the main table changes in the new ones.
- When you have different types of data in your database, such as names, dates, and pictures. 
  You could keep the string values in SQL DB (expensive), and pictures in an Azure Blob (cheap).
  
### Directory based partitioning
- Directory based shard partitioning involves placing a lookup service in front of the partioned databases.
- The lookup service knows the current partitioning scheme and keeps a map of each entity and which database shard it is stored on. 
  The lookup service is usually implemented as a webservice.
  
## Partitioning criteria
1. **Range based**:

   This type of partitioning assigns rows to partitions based on column values falling within a given range. e.g, store_id is a column of the table.
   ```SQL
   PARTITION BY RANGE (store_id) (
    PARTITION p0 VALUES LESS THAN (6),
    PARTITION p1 VALUES LESS THAN (11),
    PARTITION p2 VALUES LESS THAN (16),
    PARTITION p3 VALUES LESS THAN (21)
    )
    ```
2. **Key or hash based**:
 
   Apply a hash function to some key attribute of the entry to get the partition number. e.g. partition based on the year in which an employee was hired.
   
   ```SQL
   CREATE TABLE employees (
    id INT NOT NULL,
    fname VARCHAR(30),
    lname VARCHAR(30),
    hired DATE NOT NULL DEFAULT '1970-01-01',
    separated DATE NOT NULL DEFAULT '9999-12-31',
    job_code INT,
    store_id INT
    )
    PARTITION BY HASH( YEAR(hired) )
    PARTITIONS 4;
    ```
    
3. **List partitioning** 
4. **Round robiin** 
5. **Composite** 

## Common problems of sharding
Most of the constraints are due to the fact that operations across multiple tables or multiple rows in the same table will no longer run on the same server.

1. **Joins and denormalization**:

  Joins will not be performance efficient since data has to be compiled from multiple servers.
  Workaround: 
  Denormalize the database so that queries can be performed from a single table. But this can lead to data inconsistency.

2. **Referential integrity**:

  Difficult to enforce data integrity constraints (e.g. foreign keys).
  Workaround:
      1. Referential integrity is enforced by the application code.
      2. Applications can run SQL jobs to clean up dangling references.

3. **Rebalancing**:

  Necessity of rebalancing
    1. Data distribution is not uniform.
    2. A lot of load on one shard.



