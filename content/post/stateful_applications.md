---
title: Stateful Applications
description: What is a stateful application? Why do we need this architcture?  
date: 2021-03-08
categories:
  - "Development"
tags:
  - "Engineering"
  - "server-side"
---
What is a stateful application? Why do we need this architcture?
<!--more-->

Let us consider this scenario:
1. User signs into the application (web/mobile or any) and say lands in "overview" page of the application
2. This Overview page needs to pull summary of all their transactions 
   - This request goes to the backend & lands in server node N1
3. After seeing the summary user clicks on "balance" amount to get more dtails
   - Now, this second request needs to go to same server node N1, cause that node has been warmed up for this user - or few things are cached up for this user OR user sessions.

## User Sessions

The backend application servers [Tomcat](https://tomcat.apache.org/), [Jetty](https://www.jetty.com/), [ASP.NET](https://dotnet.microsoft.com/apps/aspnet), [Django](https://www.djangoproject.com/) etc., have a concept of “Session” or “State Management”.
A User [Session](https://en.wikipedia.org/wiki/Session_(computer_science)) is a temporary collection of objects stored in memory for a specific user with strict time to live (TTL)(eg. 30 mins). Remember JSESSIONID ?

Session based application pattern is created to improve performance by avoiding expensive queries to the Database. If we use this pattern, we need to establish a sticky behavior between the user and the backend node they are interacting with - these are also called as Stateful Applications.

### For Eg:
1. We have a cluster of application nodes — N1, N2 & N3 — all running server instances.
1. User logs into the application
1. They land in a server node — N1
1. In N1, application pulls user’s metadata — for eg. preferences, configurations etc., and stores them in a user Session object — local session(preserved in memory or high performance cache/datastore with an expiry time)
1. Hereafter, all future requests for this user is routed to same node N1 by the load balancer. This is known as sticky session. To achieve stickiness a special header or session based cookie (eg. JSESSIONID) will be used.
1. Same user session object is fetched from cache/session store (without hitting backend data layer)

![User Sessions](https://miro.medium.com/max/700/1*BB89hw15DEXVo99hWwRQlQ.png)

## Session Management - The Bad

Now, in the above example, let us say node N1 goes down for some reason, we will lose all the sessions stored in N1. So by design we keep a replica of N1:sessions in say N3.

![the bad](https://miro.medium.com/max/700/1*7mKeVKu2E7VBsB75NYIk1A.png)

If node N1 goes down, requests are routed to N3 and N3 now serves both N1 & N3. N1 recovers & new requests will go there. N3 will continue to serve users with capacity equal to two nodes.

Then application servers like ATG Dynamo brought the concept of [Session Federation](https://docs.oracle.com/cd/E26180_01/Platform.94/ATGPersProgGuide/html/s1603scenariocachingwithsessionfedera01.html) — which balances the sessions across all nodes & load balance the traffic accordingly whenever there is change to number of active nodes. They flush sessions from node to node, keeping a registry of session to node mapping in a node. That will have a replica as well. See, now this is getting complicated.

### Bloating Sessions
Developers started using this utility object (session) conveniently for storing tons of other metadata about the user — to avoid expensive DB calls. Now, sessions started to grow like kitchen sink. Imagine if you have JVM with > 50% of memory used for storing sessions.
Session OR state management was a fancy design pattern of 2000s, it is not followed anymore. So we may see this in legacy applications. Newer applications follow a simpler approach.


## Solutions
### 1. Session Repository

Instead of every node managing sessions, manage them in a central high speed cache datastore like Memcached or Redis.

![Session Repository](https://miro.medium.com/max/700/1*Cj5H9nPwpfrUqWgWexqDKA.png)

1. Now, nodes need not worry about maintaining local Sessions
1. Application Load-balancer need not route the user requests to same node N1. In other words no sticky sessions.
1. All we have to do is design the cache for resiliency — cluster with failover nodes


### 2. Go Stateless 
Removes the need for sessions altogether. For performance, better data modeling should help. If we are dealing with existing low performant infrastructure, use caching as needed.

## Benefits of Stateful Application pattern
One positive outlook is — the concept of sessions or sticky sessions could help us with Testing a feature going live. Imagine you are pushing code & you want to test the application with your request hitting the latest version of the application — ec2 instance / k8s pod. AWS [Classic ELB](https://docs.aws.amazon.com/elasticloadbalancing/latest/classic/elb-sticky-sessions.html) supports sticky sessions and so does k8s ([session affinity](https://sookocheff.com/post/kubernetes/building-stateful-services/)). However, this need not be used for managing user sessions.

## References

1. [Session Management in Java](https://medium.com/@kasunpdh/session-management-in-java-using-servlet-filters-and-cookies-7c536b40448f) — Old example. Nobody uses JSPs these days. So read for conceptual understanding.
1. [Tomcat Session Management](https://dzone.com/articles/redis-based-tomcat-session-management) using Redis

