---
layout: post
title: "Amazon Aurora databases"
subtitle: "And why I like them"
date: 2020-07-05 00:30:00 -0400
background: '/img/posts/10.jpg'
---

## Database engines
Databases are your application's long-term data store, which application servers use to provide business value. Traditionally, a database is also just a computer that stores all its data in long-term memory (i.e. not RAM).

The way that your backend application communicates with your database depends on the database engine. Modern backend frameworks come with ORMs (Object-Relational Mappers) that are compatible with most popular database engines, like PostgreSQL. Your source code uses your ORM's operations, and your ORM conveys these commands to your database.

#### Amazon Aurora
Traditionally, databases store their contents in storage volumes that are mounted onto server instances. When an incoming request is received, the server loads the relevant contents from storage, incurring latency due to input/output times, aswell as requiring memory and CPU. If you wish to use a more or less powerful database instance, you'd need to set up replication from the old to the new database, and then switch traffic over. 

However, as these storage volumes get bigger, the time required to mount them into new database instances increases to several hours. Application servers can autoscale the size of their servers the match the current workload, but databases take so long that by the time you've scaled up, your desired capacity has changed again. Therefore, you need to provision your database at maximum capacity, overprovisioning for the majority of the day. 

This was the case until somebody had the bright idea of separating a database's storage and compute power.

## Separate storage and compute power
AWS Aurora is a database engine with such a set-up, with a super-scalable S3-based storage layer, built with robustness and high availability in mind. Its performance scales flawlessly with increased I/O requests, as you'd expect from any managed service.

To extract information from this storage layer, a compute layer is still needed. This layer is controlled by the user, who can choose what instance should establish a connection with the storage layer. Note that the compute layer does not mount this storage layer, meaning that the time taken for a new compute instance to be created does not scale with the amount of data in the database. Regardless of storage or compute type, my tests have indicated that connections are created in between 4 and 5 minutes.

For the storage layer, users pay per million I/O requests, and for network traffic volume. For the compute layer, users pay per minute that they use each instance, and this is also instance-type dependent. The compute costs usually far outweigh the storage costs, but the ease with which compute instances can be added and deleted from the database creates a huge opportunity for scaling, and therefore, cost optimisation.

## Read number scaling
We've established that you can create new connections to the storage layer easily. This makes it very easy to add read replicas to the database (this is called a regional cluster of database instances). Aurora provides a write and a read endpoint for each regional cluster, and will load balance traffic between all read replicas evenly.


## Read and write size scaling
To retain a single source of truth, standard Aurora engines only allow a single write instance at a time. Therefore, if you need to scale your database's write capacity, you can create a new read replica using your desired instance size, promote it to master, and then delete the old write instance. My tests have shown that this failover happens instantly, meaning that open backend connections to the database are cut off and instantly re-opened, with minimal effects on users. Meanwhile, the storage layer scales to provide a constant performance, because it's still a managed service.

## Capacity scaling
The ability to re-provision your database's capacity within minutes allows users to avoid overprovisioning. Instead of running at rush-hour capacity during the nighttime, you can now scale down gradually in the evening, and scale up gradually in the morning, cutting your database's costs hugely. Of course, the profitability of implementing this technique depends on how variable the estimated workload is throughout the day, and how consistently and accurately you can predict required capacities.

## part 2
## Disaster recovery cost-effectiveness
Next, I'll discuss global clusters, and their cost-effective solution to disaster recovery requirements


#### Switching to Aurora
Due to the aforementioned managed nature of database communication in modern frameworks, AWS have made the switchover to Aurora as easy as it could ever be. 


## Notes on performance
- learn about second indices


## Notes on implementation
- Read and write connection management in Rails
- Fixed endpoint creation
- Cut DB connections on switching
- Terraform management is chaotic for instances - learn about ASGs
- Version limitations
- No reserved instances if youre scaling
- ASGs vs lambdas
