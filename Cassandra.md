# Cassandra

[TOC]



# Introduction

## Problems while scaling Relational Databases

### Replication: ACID is a lie

Data is replicated asynchronously, this is known as replication lag. When the client decides to write to the master it takes some time to replicate to the salve. If the client decided to do a read, it gets the old data back. Hence the data is not consistent.

![image-20210508190032962](Cassandra.assets/image-20210508190032962.png)

### Third Normal Form doesn't Scale

- Queries are unpredictable
- Takes time to execute
- Data must be denormalized
- If data > memory, you = history
- Disk seeks are the worst

To be able to query data faster, we generally create denormalized views, which essentially takes away any benefits we could reap from having the data in the normalized form

### Sharding is a Nightmare

- Data is all over the place
- No more joins
- No more aggregations
- Denormalize all the things
- Querying seconding indexes requires hitting every shard
- Adding shards requires manually moving the data
- Schema changes - Apply schema changes to all of the shares in the system, both master and slave

### High Availability.. not really

- Master failover.. who's responsible? 
  - Another moving part... - Who will bring up the master if it is crashed, if there is an automated service, who will bring up the automated service when that itself is down
  - Bolted on hack
- Multi-DC is a mess - In case of multiple data centers it is hard to manage manually
- Downtime is frequent
  - Change database settings - needs restart
  - Driver, power supply failures
  - OS updates

**Goad here is to achieve a higher availability than what a master slave architecture offers**

### Summary of Failure

- Scaling is a pain
- ACID is naive at best
  - You aren't consistent
- Re-sharding is a manual process
- We're going to denormalize for performance
- High availability is complicated, requires additional operation overhead

### Lessons Learned

- Consistency is not practical
  - So we give it up
- Manual sharding & rebalancing is hard - meeds writing a lot of code
  - So let's build in - push the responsibility to the cluster
- Every moving part makes systems more complex
  - So let's simplify our architecture - no more master/ slave
- Scaling is expensive - vertical scaling is expensive, only use commodity hardware
- Scatter/ gather no good - We want to achieve data locality, so that there is no extra latency in scaling the entire cluster to retrieve records. This is done with better data modeling

## Cassandra Overview

### What is Apache Cassandra?

- Fast distributed database
- High availability
- Linear scalability
- Predictable performance
- NO single point of failure
- Multi - data center
- Commodity Hardware
- Easy to manage operationally
- Not a drop in replacement for RDBMS - **Have to design the application around Cassandra's data model, we cannot just simply follow the same data model in Cassandra**

### Hash Ring

- No master/ slave/ replica sets
- No config servers, zookeeper
- Data is partitioned around the ring
- Data is replicated to RF=N servers
- All nodes hold data and can answer queries (both read & writes)
- Location of data on ring is determined by partitioned key

![image-20210508201800210](Cassandra.assets/image-20210508201800210.png)



**When we create a table in Cassandra we assign a partition key, the value of the partition key is run through a consistent hashing function and the depending on the output it is determined in which node the data will be stored** 

### CAP Tradeoffs

- Impossible to be both consistent and highly available during a network partition
- Latency between data centers also makes consistency impractical
- Cassandra chooses Availability & Partition tolerance over Consistency

### Replication

How many copies of each piece of data should there be on the cluster

- Data is replicated automatically
- You pick number of servers
- Called "replication factor" or RF
- Data is ALWAYS replicated to each replica
- If a machine is down, missing data is replayed via a hinted handoff

![image-20210508202431481](Cassandra.assets/image-20210508202431481.png)



### Consistency Level

- Per query consistency - for both read and write
- Replicated factor is honored irrespective of the Level of the Consistency
- How many replicas for query to respond to OK
  - ALL - All replicas
  - QUORUM - Majority, so in case of replication of 3, this implies 2 out of 3 (51% or greater)
  - ONE - One replica

**Consistency level has impact on how fast we can read or write data, it is also going to impact the availability, if the consistency level is set to all, even a single node failing can make the cluster unavailable. With Level 1, the cluster will be highly available**

### Multi DC

- Typical usage: clients write to local DC, replicates async to other DCs
- Replication factor per keyspace per datacenter
- Datacenters can be physical or logical

![image-20210508203110903](Cassandra.assets/image-20210508203110903.png)

**Logical datacenters are useful for segregating OLTP queries which need fast responses to OLAP queries, hence the data will be un sync, but Analytics queries will not impact the OLTP queries**



## Cassandra Internals and Choosing a Distribution

### The Write Path

- Writes are written to any node in the cluster (Coordinator) - **Whatever node you happen to be talking to in that request**, for that write request this node does the coordination - (**like a temp master**)
- Writes are written to commit log, then to memtable  - **commit log is an append only datastructure, it is going to be sequential IO** , then it merges to the memtable, which is an in-memory representation of the table
- Every write includes a timestamp
- Memtable flushed to disk periodically (sstable) - **When the memtable starts to exceed the memory, we flush it to disk as sstable** 
- New memtable is created in memory
- Deletes are a special write case, called a "tombstone" -  **Cassandra never does any updates in-place or deletes in-place, the sstables are immutable, commit log is immutable. So when we delete something Cassandra just marks that there is no data here, a tombstone has a timestamp and it indicates that there is no data at that particular timestamp for which we deleted a record**

![image-20210508204455475](Cassandra.assets/image-20210508204455475.png)



### What is an SSTable?

- Immutable data file for row storage 
- Every write includes a timestamp of when it was written
- Partition is spread across multiple SSTables
- Same column can be in multiple SSTables
- Merged through compaction, only latest timestamp is kept
- Deletes are written as tombstones
- Easy backups!

![image-20210508205032133](Cassandra.assets/image-20210508205032133.png)



### The Read Path

- Any server may be required, it acts as the coordinator
- Contacts nodes with the requested key
- On each node, data is pulled from SSTables and merged
- Consistency<ALL performs read repair in background (read_repair_chance) - **This the chance that whenever you do a read, all the nodes have the latest updated data**. The default value is 10%, so for 10% of the reads Cassandra is going to attempt to keep all of the data in sync.

![image-20210508225802934](Cassandra.assets/image-20210508225802934.png)

**The storage type SSD vs HDD is going to have a huge impact on the read performance**

### Picking a distribution

#### Open Source

- Perfect for hacking
- Latest, bleeding edge features

#### DataStax Enterprise

- Integrated Multi-DC Search
- Integrated Spark for Analytics
- Focused on stable releases for enterprise
- Additional QA



## CQL

### Keyspaces

- Top-level namespace/container
- Similar to a relational database schema

```CQL
CREATE KEYSPACE killrvideo
WITH REPLICATION = {
	'class': 'SimpleStrategy',
	'replication_factor': 1
};
```

- Replication parameters required

### USE

- USE switches between keyspaces

```cql
USE killrvideo;
```

