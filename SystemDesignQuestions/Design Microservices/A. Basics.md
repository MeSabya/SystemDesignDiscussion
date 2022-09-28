![image](https://user-images.githubusercontent.com/33947539/192696473-a132e858-99bc-4f78-8bac-c71eb5f93fec.png)

![image](https://user-images.githubusercontent.com/33947539/192696611-0e402256-5c29-4d96-8ff6-c34bb2ab2e5a.png)

## Benefits of Microservices Architecture

### Agility
One of the most important characteristic of microservices is that because the services are smaller and independently deployable,
it’s easier to manage bug fixes and feature releases. 
You can update a service without re-deploying the entire application, and roll-back an update if something goes wrong. 
In monolithic applications, if a bug is found in one part of the application, it can block the entire release process. 
So new features was waiting for a bug fix to be integrated, tested, and published.

### Small, focused teams

### Small and Separated Code Base

### Right tool for the job

In traditional layered architectures, an application typically shares a common stack, with a large relational database supporting the entire application. This approach has several challenges for example every component of an application must share a common stack, data model and database even if there is a clear, better tool for the job for certain modules. It’s really frustrating for developers who are aware that a better, more efficient way to build these components is available. Also developers is frustrating when the application stack is too old and can’t apply new best practices on their projects.

### Fault Isolation
If one of your microservice becomes unavailable, it won’t affect the entire application. Of course you should design your microservices are fault tolerance and handle faults correctly for example by implementing retry and circuit breaking patterns. Even failures happened, if you fix that failures without any business affect, you customer will always happy.

### Scalability
scaling is very easy with using an container orchestrator tool like Kubernetes, you can pack a higher volume of services onto a single host, which allows for more efficient utilize of hardware resources.

### Data isolation
Since microservices following the database-per-service patterns, databases are separated with each other according to microservices design. So it gets easier to perform schema updates, because only a single database is affected. In a monolithic application, schema updates can become very challenging, and risky.

## Challenges of Microservices Architecture

### Complexity
Each service is simpler, but the entire system is more complex. Even deployments can be complicated for hundreds of services deploy different times hard to manage versions.
microservices communication is hard topic and need to have a strategy to manage inter-service communications between server even different geo-locations.

### Network problems and latency
If we call chain of services for particular request, this will increate latency problems and need correct design to APIs for proper communication. In order to avoid chatty API calls. You need to consider asynchronous communication patterns like message broker systems.

### Data Integrity
Microservice has its own data persistence. So data consistency can be a challenge. Mostly we should follow eventual consistency where possible. But transactional operations are always will be challenging.



 
