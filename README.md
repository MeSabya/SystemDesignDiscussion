# SystemDesign Learning Path

***Step1 Finish the Basics needed for System Design:***

- Database
   DB basics:
   - [ ] [How indexing Works internally](https://www.pankajtanwar.in/blog/how-database-indexing-actually-works-internally)
   - [ ] Relational database management system (RDBMS)
      - Master-slave replication
      - Master-master replication
      - Federation
      - Sharding [How Sharding works](https://medium.com/@jeeyoungk/how-sharding-works-b4dec46b3f6) 
      - Denormalization
      - SQL tuning
   - [ ] NoSQL
      - Key-value store
      - Document store
      - Wide column store
      - Graph Database
      - SQL or NoSQL

- Concurrency basics: threads, processes, threading in the language you know. Locks , mutex etc.
  - [ ] Pyhon concurrency concepts and problems: https://www.educative.io/courses/python-concurrency-for-senior-engineering-interviews

- A basic idea of how a basic web architecture is: say 
  - [ ] load balancers 
      1. [Details of LoadBalancing](https://medium.com/geekculture/load-balancing-da0bde7882f1)
      2. [Is client side load balancing a good idea](https://www.pankajtanwar.in/blog/system-design-is-client-side-load-balancing-a-good-idea)
  - [ ] [proxy Vs Reverse Proxy](https://www.pankajtanwar.in/blog/proxy-vs-reverse-proxy-using-a-real-life-example)
  - [ ] servers, 
  - [ ] Database servers, 
  - [ ] caching servers, 
  - [ ] precompute, 
  - [ ] logging big data etc. 
  - [ ] DNS
  - [ ] Horizontal scaling
  - [ ] Web server (reverse proxy)
  - [ ] API server (application layer)
  - [ ] Cache
  - [ ] Consistency patterns
  - [ ] Availability patterns
  - [ ] [Distributed Caching](https://medium.com/rtkal/distributed-cache-design-348cbe334df1)
  
  
  Just know broadly what is each layer for.
  
  **References**:
   - https://github.com/donnemartin/system-design-primer/blob/master/solutions/system_design/web_crawler/README.md
   - https://vivek-singh.medium.com/system-design-cheat-sheet-318ba2e34723
   - https://www.educative.io/module/lesson/grokking-system-design-interview/7DWqV203YWO
    
***Step2 Learn how to approach System design questions:***
   - [How To Approach System Design Problems] (https://medium.com/enjoy-algorithm/how-to-approach-system-design-interview-4d8689813173)

***Step3 System Design Patterns:***
   - [System Design Patterns](https://www.educative.io/module/lesson/grokking-system-design-interview/YMEMlvz5jGO)
   Here is the list of patterns we will be discussing:
      - Bloom Filters
      - Consistent Hashing
      - Quorum
      - Leader and Follower
      - Write-ahead Log
      - Segmented Log
      - High-Water mark
      - Lease
      - Heartbeat
      - Gossip Protocol
      - Phi Accrual Failure Detection
      - Split-brain
      - Fencing
      - Checksum
      - Vector Clocks
      - CAP Theorem
      - PACELEC Theorem
      - Hinted Handoff
      - Read Repair
      - Merkle Trees   
      
***Step4 System Designing Important Questions***

- [ ] Designing a URL Shortening service like TinyURL 

- [ ] Designing Instagram

- [ ] Designing Dropbox
  1. https://www.pankajtanwar.in/blog/system-design-how-to-design-google-drive-dropbox-a-cloud-file-storage-service
  2. https://www.educative.io/module/lesson/grokking-system-design-interview/N0qRAomMJqz

- [ ] [How does Github store millions of repo and billions of files?](https://www.pankajtanwar.in/blog/how-does-github-store-millions-of-repo-and-billions-of-files)
- [ ] [Facebook Systemdesign](https://systemdesignprep.com/facebook)

- [ ] [Designing Facebook’s Newsfeed](https://systemdesignprep.com/newsfeed)

- [ ] Designing Facebook Messenger

- [ ] [Designing Youtube or Netflix](https://systemdesignprep.com/youtube)

- [ ] Designing Typeahead Suggestion

- [ ] Designing an API Rate Limiter
  1. https://www.educative.io/module/lesson/grokking-system-design-interview/q2vgmDp6KL2
  2. https://medium.com/@saisandeepmopuri/system-design-rate-limiter-and-data-modelling-9304b0d18250

- [ ] Designing Twitter Search 
      1.(https://www.educative.io/module/lesson/grokking-system-design-interview/YVRjEyQYzRW)
      2.(https://levelup.gitconnected.com/work-through-my-solution-to-a-system-design-interview-question-a8ea4b60513b)
- [ ] Design the Twitter timeline and search 
      1.(https://github.com/donnemartin/system-design-primer/tree/master/solutions/system_design/twitter)

- [ ] [Design a key-value cache to save the results of the most recent web server queries](https://github.com/donnemartin/system-design-primer/tree/master/solutions/system_design/query_cache)

- [ ] [Design the data structures for a social network](https://github.com/donnemartin/system-design-primer/tree/master/solutions/system_design/social_graph)

- [ ] Designing a Web Crawler 
  1. https://github.com/donnemartin/system-design-primer/tree/master/solutions/system_design/web_crawler
  2. https://www.educative.io/module/lesson/grokking-system-design-interview/BnEymP49kpx

- [ ][Flight booking Website System Design](https://ankita4priya.medium.com/flight-booking-website-app-system-design-899c626a6ee6) 
- [ ] Designing Yelp or Nearby Friends

- [ ] Designing Uber backend

- [ ] Designing Ticketmaster

- [ ] [Logging Infrastructure System Design](https://www.learnsteps.com/logging-infrastructure-system-design/)

- [ ] Design a file storage service

- [The Most important one for the System Design] (https://github.com/shashank88/system_design)
- [Confusing Terms in System Design] (https://medium.com/swlh/confusing-terms-in-system-design-concurrency-vs-parallelism-performance-vs-scalability-proxy-vs-e3717b3bd81e)
- [Scalable Web Architechture and Distributed Systems] (http://www.aosabook.org/en/distsys.html)
- 
