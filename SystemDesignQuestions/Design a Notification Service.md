###### Reference: https://nayak-saabyasachi.medium.com/design-a-notification-system-c9c47ed5b9af

Summary of Notification Service:
================================

1. Functional requirement 
--------------------------
   - NS should able to send notifications to all subscribers.
   - NS should be able to prioritize the notifications.
   - NS should be able to send simple/bulk notifications
   - Email, SMS , push notifications on mobile 
   
2. NFR 
---------------------------
   - latency: NS should have low latency to deliver OTPs.  
   - Available - should be highly avl.
   - scalable - To support higher no of customers.
   
3. HLD
----------------------------
   - service ---> LB --> NS --> Msg Q --> Email , Msg , APN services --> user devices

4. NS Database to use 

5. Common services in NS. (Schedule service, validation service, prioritization service, Rate limiter)

5. Why we need Message Queue Service.
    -- Module decoupling 
    -- Msg prioritization
    -- Reliablity We should have a retry mechanism , DLT queue can be used for this.

6. Rate limiter 

7. How to ensure at most once delivery.

8. Detailed design.	
