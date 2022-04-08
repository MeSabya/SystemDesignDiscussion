# Scaling a relational database

**There are many techniques to scale a relational database: **

  1. master-slave replication, 
  2. master-master replication, 
  3. federation, 
  4. sharding, 
  5. denormalization, and 
  6. SQL tuning.


👉 *Replication usually refers to a technique that allows us to have multiple copies of the same data stored on different machines.*

👉 *Federation (or functional partitioning) splits up databases by function.*

👉 *Sharding is a database architecture pattern related to partitioning by putting different parts of the data onto different servers and the different user will access different parts of the dataset*

👉 *Denormalization attempts to improve read performance at the expense of some write performance by coping of the data are written in multiple tables to avoid expensive joins.*

## Sharding and Replication
Defined in a Separate section ..please follow the same.

## Federation:

Federation (or functional partitioning) splits up databases by function. For example, instead of a single, monolithic database, you could have three databases: forums, users, and products, resulting in less read and write traffic to each database and therefore less replication lag.

![image](https://user-images.githubusercontent.com/33947539/162372590-11a93852-4954-4f10-907e-ed3b9b893529.png)

## Denormalization

*Denormalization attempts to improve read performance at the expense of some write performance. Redundant copies of the data are written in multiple tables to avoid expensive joins.*

```
Once data becomes distributed with techniques such as federation and sharding, managing joins across data centers further increases complexity. Denormalization might circumvent the need for such complex joins.
```

## References:
https://levelup.gitconnected.com/how-to-design-a-system-to-scale-to-your-first-100-million-users-4450a2f9703d






