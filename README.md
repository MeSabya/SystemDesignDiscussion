# SystemDesign Learning Path

## Step1 Finish the Basics needed for System Design:***

- Database
   DB basics:
   - [ X] [Database Indexing](https://github.com/MeSabya/SystemDesignDiscussion/blob/main/6.%20DatabaseIndexing.md)
   - [ ] Scaling a relational database 
      - [Master-slave replication](https://github.com/MeSabya/SystemDesignDiscussion/blob/main/2.%20data-replication.md#single-leader-replication-active-passive-or-master-slave-replication)
      - [Master-master replication](https://github.com/MeSabya/SystemDesignDiscussion/blob/main/2.%20data-replication.md#multi-leader-replication)
      - Federation
      - Sharding [How Sharding works](https://medium.com/@jeeyoungk/how-sharding-works-b4dec46b3f6) 
      - Denormalization
      - SQL tuning
      - [SQL transaction isolation levels]()
   - [ ] NoSQL
      - Key-value store
      - Document store
      - Wide column store
      - SQL or NoSQL

- Concurrency basics: threads, processes, threading in the language you know. Locks , mutex etc.
  - [ ] Pyhon concurrency concepts and problems: https://www.educative.io/courses/python-concurrency-for-senior-engineering-interviews

- A basic idea of how a basic web architecture is: say 
  - [ ] load balancers 
      1. [Details of LoadBalancing](https://medium.com/geekculture/load-balancing-da0bde7882f1)
      2. [Is client side load balancing a good idea](https://www.pankajtanwar.in/blog/system-design-is-client-side-load-balancing-a-good-idea)
  - [ ] [proxy Vs Reverse Proxy](https://www.pankajtanwar.in/blog/proxy-vs-reverse-proxy-using-a-real-life-example)
  - [ ] [Consistency patterns](https://github.com/MeSabya/SystemDesignDiscussion/blob/main/SystemDesignBasics/1.3ConsistencyPatterns.md)
  - [ ] Availability patterns
  - [ ] [Distributed Caching](https://medium.com/rtkal/distributed-cache-design-348cbe334df1)
  - [ ] [Consistency Hashing](https://github.com/MeSabya/SystemDesignDiscussion/blob/main/3.ConsistentHashing.md)
  - [ ] Load Balancers 
  - [ ] Caching / Memcache
  - [ ] Distribution of Data
  - [ ] Speed / space / time trade-offs
  - [ ] Pagination
  - [ ] API’s
  - [ ] Partitioning
  - [ ] Throughput
  - [ ] [CAP theorem](https://github.com/MeSabya/SystemDesignDiscussion/blob/main/3.%20CAPTheoram.md#what-is-cap-theorem)
  - [ ] Client and server architecture
   
  
  **References**:
   - https://github.com/donnemartin/system-design-primer/blob/master/solutions/system_design/web_crawler/README.md
   - https://vivek-singh.medium.com/system-design-cheat-sheet-318ba2e34723
   - https://www.educative.io/module/lesson/grokking-system-design-interview/7DWqV203YWO
    
## Step2 Learn how to approach System design questions:
   - [How To Approach System Design Problems] (https://medium.com/enjoy-algorithm/how-to-approach-system-design-interview-4d8689813173)

## Step3 System Design Patterns:
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
      
## Step4 System Designing Important Questions

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

- [x] Designing an API Rate Limiter [Completed on 21/12/2021]  ✔
  1. https://www.educative.io/module/lesson/grokking-system-design-interview/q2vgmDp6KL2
  2. https://medium.com/@saisandeepmopuri/system-design-rate-limiter-and-data-modelling-9304b0d18250
  3. https://medium.com/wineofbits/designing-a-distributed-rate-limiter-deep-dive-76d7e8d8452d

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

### Top 10 System Design Interview Questions
1. Design a chat service
2. Design a ride-sharing service
3. Design a URL shortening service
4. Design a social media service
5. Design a social message board
6. Design a file storage service
7. Design a video streaming service
8. Design an API Rate Limiter
9. Design a proximity server
10. Design a Type-Ahead service

### More Design Questions 
1. How Do You Design a Traffic Control System?
2. How Do You Design a Website Like Pastebin?
3. How Would You Create Your Own Instagram?
4. How Do You Design a Twitter Clone?
5. How to Design an ATM Machine?
6. How Do You Design an API Rate Limiter?
7. How Do You Design Twitter Search?
8. How to Design a Web Crawler Like Google?
9. How to Design BookMyShow?
10. How Do You Design an Application Like Airbnb?
11. How Do You Design an Elevator of the Lift System?
12. How Do You Design Order Management System
13. Design of Distributed job scheduler.
14. Design of Read Receipts mechanism in Whatsapp.
15. Design of Bus scheduling system.
16. Design of Ticket booking system.
17. Design of System to handle flash sales.
18. Design of Netflix.
19. Design of Access Management System.
20. Design of Multiplayer game.
21. Design of Railways Cloak Room.
22. Design of Payment mechanism.
23. Design of Price automation system.
24. Design of Voice assistant used in mobile.
25. Design of Event booking system.
26. Design of MP3 player.
27. Design of File conversion tool.

##### Reference: 
[Top 10 System Design Questions](https://medium.com/geekculture/top-10-system-design-interview-questions-10f7b5ea123d)
[Approaching System Design Questions](https://medium.com/enjoy-algorithm/how-to-approach-system-design-interview-4d8689813173)
[Best System Design Cheat Sheet](https://vivek-singh.medium.com/system-design-cheat-sheet-318ba2e34723)
[Advanced System Design Interview](https://www.educative.io/courses/grokking-adv-system-design-intvw/xoEXr9614RB)


### System Design Template 

![image](https://user-images.githubusercontent.com/33947539/148498041-b60a7594-1398-451d-812b-9c49b626e6d1.png)

