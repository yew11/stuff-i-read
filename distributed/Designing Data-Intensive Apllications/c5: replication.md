## Replication 
replication means keep a copy of the same data on multiple machines that are connected via a network. 

The difficulty in replication is handling changes of replicated data. 

Three Algorithms for replicating changes b/w nodes, almost all use one of these three:
1. [single-leader](#leaders-and-followers)
2. [multi- leader](#multi-leader-replication)
3. leaderless 

### Leaders and Followers
With multiple replicas, how do we ensure that all the data ends up on all the replica? 

The most common approach: 
1. when clients want to write to the db, they must send their requests to the leader. which first writes the new data to its local storage. 
2. When the leader writes new data to its local storage, it also sends the data changes to all followers as part of a _replication_ or _change stream_. 
3. when clients want to read from db, it can query any, but writes are only accepted on the leader. 

Synchronous vs. Asynchronous replication 
![Screen Shot 2020-01-07 at 8 23 04 PM](https://user-images.githubusercontent.com/25108604/71945138-90562000-318b-11ea-8a9b-5c48d85a66e0.png)

Follower 1 is synchronous, while the follower 2 is asynchronous.  

**About synchronous**: 
1. followers is guaranteed to have an up-to-date copy of the data. If the leader fails, the data is still available on the followers.  
2. Disadvantage is write cannot proceed if one follower fails. 

For above reason, it is impractical for all followers to be synchronous, any outage, would cause the whole system to grind to halt. 

In practice, if you enable synchronous replication on a database, it usually means that one of the followers is synchronous, and the others are asynchronous. If the synchronous follower becomes unavailable or slow, one of the asynchronous followers is made synchronous. This guarantees that you have an up-to-date copy of the data on at least two nodes: the leader and one synchronous follower.
This is called **semi-synchronous**

Often, leader-based replication is configured to be completely **asynchronous**, this means a write is not guaranteed to be durable. The advantage is the leader can continue processing writes, even if all of its followers have fallen behind.  

### Setting up new followers 
Simply copying data files from one node to another not sufficient: clients are constantly writing to the db, the data is always in flux. So the process is: 
+ Take a consistent snapshot of leader's db at some point in time. 
+ Copy the snapshot to the new follower node
+ The follower connects to the leader and requests all the data changes that have happened since the snapshot was taken. This requires that the snapshot is associated with an exact position in the leader's replication log. 
+ When the follower has processed the backlog of data changes since the snapshot, now it has caught up. 

### Handling Node outages
Goal: keep the system as a whole running despite individual node failures, and keep the impact of a node outage as small as possible. 

**Follower failure: Catch-up recovery**
On its local disk, each follower keeps a log of data changes it has received from the leader. If crashes, the follower can recover easily from its logs. 

When the follower is online, it can requests data changes during the downtime. 

**Leader failure: Failover**
On of the followers needs to be promoted to be the new leader: clients need to write data to the new leader, and other followers need to start consuming data changes from the new leader. This is called **failover**
1. Determine if leader has failed: if the leader timed-out
2. Choose a new leader: the replica with the most up-to-date changes from the old leader. 
3. Reconfigure the system to use the new leader. When the old leader comes back, it needs to be a follower. 

Failover is fraught with things that can go wrong:
1. If asynchronous replication is used, the new leader may not have received all the writes from the old leader before it failed. The most common solution is for the old leader's unreplicated writes to be discarded.
2. Discarding writes is especially dangerous if other storage systems outside of the database need to be coordinated with the database contents. 
3. Split brain, both nodes can think they are the leader. So shut down one node if new leaders are detected. 
4. What is the right timeout before the leader is declared dead? 

### Replication logs implementation 
**Statement-based replication**

the leader logs every write request that it executes and sends that statement log to its followers (Every INSERT, UPDATE, DELETE). 

problems: 
1. any statement that calls a nondeterministic function, such as `NOW()` or `RAND()`, is likely to generate a different value. 
2. If the statement use an auto incrementing column, or they depend on the existing data in the db, they must be executed in the **same order**
3. statement that have side effects (triggers, stored procedures, etc...),  may result in different side effects.

It is possible the leader replace nondeterministic functions with a fixed value, but there are so many edges cases to consider.  

**Write-ahead log shipping**
Leader: log is an append-only sequence of bytes containing all writes to the db. When the follower process the log, it builds a copy of the exact same data structure as found on the leader. 

problem:
log describes the data on a very low level: a WAL contains details of which bytes were changed in which disk blocks: This makes replica closely coupled to the storage engine. Fucked if the db changes its storage format. 

**Logical (row-based) log replication**

use different  log formats for replication and for the storage engine, which allows the replication log to be decoupled from the storage engine internals. 

A logical log for a relational db is usually a sequence of records at the granularity of a row. 
1. For an inserted row, log contains the new values of all columns. 
2. For a deleted row, contains information to uniquely identify the row (primary key or all the columns).
3. Updated row, new values of all columns. 

**Trigger-based replication**
A trigger lets you register custom application code that is automatically executed when a data changes occurs in a db system. But larger overhead and prone to bugs.  




### Problems with Replication lag

Leader-based replication is great for read heavy applications. 

If an application reads from an asynchronous follower, it may see outdated data. If we wait a while, all the follower will have the same data, _Eventual Consistency_

if the system is operating near capacity or if there is a problem in the network, the lag can easily increase. 

#### Reading your own writes
![Screen Shot 2020-01-09 at 8 13 53 PM](https://user-images.githubusercontent.com/25108604/72120149-94657780-331c-11ea-9a7e-a3c23beffcee.png)

In this case, we need _read-after-write consistency_ or _read-your-writes consistency_. If only guarantee for the current writer, not guaranteed for other users. How do we implement? 
1. When reading something that user may have modified. Read it from the leader. otherwise, read it from the follower. So somehow know what have been modified by the user (user's profile, for example)
2. If most things are editable by the user, other criteria decide whether to read from the leader: Keep track of the time last updated. if within some time range, read from leader. 
3. If replicas are distributed across multiple datacenters, Any request that needs to be served by the leader must be routed to the datacenter that contains the leader. 

But, what if user have multiple devices, additional issues to consider. 
1. remembering the timestamp becomes more difficult. code running on one device doesn't know updated happened to other device. This metadata will need to be centralized. 
2. Replicas across different datacenters, different devices could be routed to different datacenter. 

#### Monotonic Reads
When reading asynchronous followers a user can see things _moving backward in time_.
This happens when a user make multiple reads from different replicas(say two).  

If the first reads returns something just been written, but the second reads happens to read from a datacenter that hasn't been written to, it could be very confusing for the user.

**Monotonic reads** guarantee this does not happen, it's a lesser guarantee than strong consistency, 
but a stronger guarantee than eventual consistency. It make sure that each user always makes their reads from the same replica. Could be chosen based on a hash of the user ID. 

#### Consistent Prefix reads
consistent prefix reads: if a sequence of writes happens in a certain order, then anyone reading those writes will see them appear in the same order.

This problem has in partitioned databases. In many distributed dbs, different partitions operate independently. No global ordering of writes. All write to the same partition? Not so efficient  

## Multi-Leader Replication 
_master-master_ or _active/active_ replication 

Have  a database with replicas in several different datacenters. One leader per datacenter. 

+ Performance: 
In a single-leader configuration, every write must find the one leader, may increase latency to writes. 
In a multi-leader configuration, every write can be processed in the local datacenter and is replicated asynchronously to other datacenters. network delay is hidden from users. 

+ Tolerance of datacenter outages: 
Single: A leader fails, failover can promote a follower in another datacenter to be leader. 
Multi: each datacenter can continue operate independently.  

Downside for multi-leader: same data may be concurrently modified in two different datacenters, and those weri

User case - Collaborative editing: 

When one user edits a document, the changes are instantly applied to their local replica and asynchronously replicated to the server and any other users who are editing the same document.

Obtain a lock on the document before the user can edit it. This collaboration model is equivalent to single-leader replication with transactions on the leader.

However, for faster collaboration, you may want to make the unit of change very small and avoid locking. This requires conflict resoltion. 

### Handling Write Conflicts

![Screen Shot 2020-01-10 at 8 57 59 PM](https://user-images.githubusercontent.com/25108604/72197901-2d16f880-33ec-11ea-88b5-2692b1913770.png)

Each user's change is successfully applied to their local leader. However, when the changes are asynchronously replicated, a conflict is detected. This does not occur in a single-leader db. 

**Synchronous versus asynchronous conflict detection**

