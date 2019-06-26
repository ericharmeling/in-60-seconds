@snap[midpoint span-100]
## CockroachDB
### A Brief Overview
@snapend

---
## Agenda

@snap[midpoint text-left span-100]
@ul[spaced text-25]
- **What is CockroachDB?**
- **How is data stored?**
- **How is data replicated and distributed?**
- **How do reads and writes behave in CockroachDB?**
- **How does CockroachDB tolerate failures?**
- **How does CockroachDB guarantee consistency?**
@ulend
@snapend

---
## Agenda

@snap[midpoint text-left span-100]
@ul[spaced text-25]
- **What is CockroachDB?**
- How is data stored?
- How is data replicated and distributed?
- How do reads and writes behave in CockroachDB?
- How does CockroachDB tolerate failures?
- How does CockroachDB guarantee consistency?
@ulend
@snapend

---

## What is CockroachDB?
### FAQ Answer

@snap[midpoint span-100]
"*CockroachDB is a distributed SQL database built on a transactional and strongly-consistent key-value store.*" - [FAQs: What is CockroachDB?](https://www.cockroachlabs.com/docs/v19.1/frequently-asked-questions.html#what-is-cockroachdb)
@snapend

---

## What is CockroachDB?
### A distributed, "NewSQL" database

@snap[midpoint span-40]
![](assets/img/crdb.png)
@snapend

@snap[south span-100]
@ul[spaced]
- A **node** is an instance of CockroachDB.
- A **cluster** is a group of connected nodes that acts as a single application.
@snapend

---

## What is CockroachDB?
### A distributed, "NewSQL" database

@snap[midpoint span-40]
![](assets/img/crdb.png)
@snapend

@snap[south span-100]
@ul[spaced]
- A SQL client.
- A key-value store.
- Some other components that handle distributing, replicating, and storing data in a way that guarantees **ACID** properties.
@snapend

---

## What is CockroachDB?
### ACID Guarantee

@snap[midpoint span-100]
@ul[spaced]
- **A**tomicity (*Transactions happen or they don't.*)
- **C**onsistency (*Data is always in a valid state, across all locations.*)
- **I**solatation (*Transactions are separate, and strongly serializable.*)
- **D**urability (*The data is permanently and durably recorded.*)
@ulend
@snapend

---

## What is CockroachDB?
### Architecture

@snap[midpoint span-100]
@ul[spaced]
- **SQL** Layer: "*Translate client SQL queries to KV operations.*"
- **Transactional** Layer: "*Allow atomic changes to multiple KV entries.*"
- **Distribution** Layer: "*Present replicated KV ranges as a single entity.*"
- **Replication** Layer: "*Consistently and synchronously replicate KV ranges across many nodes. This layer also enables consistent reads via leases.*"
- **Storage** Layer: "*Write and read KV data on disk.*"
@ulend
@snapend

---
## Agenda

@snap[midpoint text-left span-100]
@ul[spaced text-25]
- What is CockroachDB?
- **How is data stored?**
- How is data replicated and distributed?
- How do reads and writes behave in CockroachDB?
- How does CockroachDB tolerate failures?
- How does CockroachDB guarantee consistency?
@ulend
@snapend

---

## Storage
### How is data stored in CockroachDB?

@snap[west span-100]
@ul[spaced]
- **SQL interface**: Users access data in CockroachDB as entries in rows and columns of a table, with SQL statements. From the SQL perspective, data is represented as being in one place. The machine through which a user accesses the SQL interface is referred to the **gateway node**.
- **Key-value store**: Under the hood, data are stored in partitions ("ranges") up to 64 MiB in size of key-value pairs in a key-value store. These ranges are replicated and distributed to multiple nodes.
@ulend
@snapend

---

## Storage
### Key-value store

@snap[midpoint span-100]
@ul[spaced]
- CockroachDB stores data, including table data, indexes, and metadata in a key-value store (RocksDB).
- For table data, each key in the key-value store is a unique ID based on the table ID, and the primary key.
- Each value in the key-value store is the value of the row for the corresponding unique key.
- Indexes and metadata are stored a little differently... but still as key-value pairs in the key-value store.
@ulend
@snapend

---

## Storage
### Key-value store

@snap[east text-right span-100]
![](assets/img/keyspace1.png)
@snapend

@snap[west text-left span-100]
The "keyspace" can be represented as a monolothic, sorted map.
@snapend

---

## Storage
### Key-value store

@snap[east text-right span-100]
![](assets/img/keyspace2.png)
@snapend

@snap[west text-left span-100]
This keyspace is partitioned into ranges.
@snapend

---

## Storage
### Key-value store

@snap[east text-right span-100]
![](assets/img/keyspace3.png)
@snapend

@snap[west text-left span-60]
These ranges are replicated and distributed to nodes.
@snapend

---

## Agenda

@snap[midpoint text-left span-100]
@ul[spaced text-25]
- What is CockroachDB?
- How is data stored?
- **How is data replicated and distributed?**
- How do reads and writes behave in CockroachDB?
- How does CockroachDB tolerate failures?
- How does CockroachDB guarantee consistency?
@ulend
@snapend

---

## Replication & Distribution
### How is data replicated and distributed in CockroachDB?

@snap[midpoint span-100]
@ul[spaced]
- Data in the key-value store is partitioned into **ranges** of up to 64 MiB.
- Each range is replicated and distributed to a default minimum of three nodes on a cluster.
- When data in a range grow larger than 64 MiB, the range is split, replicated, and distributed across the cluster.
- It's very easy to scale CockroachDB horizontally. Just add new nodes to a cluster, and it will rebalance loads automatically.
@ulend
@snapend

---

## Replication & Distribution
### Leaseholders

@snap[midpoint span-100]
@ul[spaced]
- For each range, there is a replica that holds a "range lease". This replica is known as the **leaseholder** (and its node is the **leaseholder node**). This replica manages the read and write requests for its range.
- When a user submits a SQL statement, the gateway node identifies the leaseholder for the range of interest, and sends the read or write request to the leaseholder.
- We will discuss reads and writes in more detail later...
@ulend
@snapend

---

## Replication & Distribution
### Consensus with Raft

@snap[midpoint span-100]
@ul[spaced]
- CockroachDB uses the Raft consensus algorithm to guarantee that data is consistent across replicas.
- Raft groups replicas of the same range into a **Raft group**. There is one Raft group per range.
- Each group has a single **leader**. All other replicas are **followers**. The Raft leader is usually also the leaseholder.
- Each replica holds a **Raft log**, which contains a time-ordered log of writes to its range that the majority of replicas have agreed on.
@ulend
@snapend

---

## Replication & Distribution
### Raft Elections

@snap[west span-50]
@ul[spaced]
- The leader periodically distributes the latest log of writes to the followers in the group.
- If a follower does not hear from the leader within a time period, that follower starts a new election.
@ulend
@snapend

@snap[east span-50]
![](assets/img/raft.png)
[Ongaro and Ousterhout (2014)](https://www.usenix.org/system/files/conference/atc14/atc14-paper-ongaro.pdf)
@snapend

---

## Agenda

@snap[midpoint text-left span-100]
@ul[spaced text-25]
- What is CockroachDB?
- How is data stored?
- How is data replicated and distributed?
- **How do reads and writes behave in CockroachDB?**
- How does CockroachDB tolerate failures?
- How does CockroachDB guarantee consistency?
@ulend
@snapend

---

## Reading & Writing
### How do reads work in CockroachDB?

@snap[midpoint span-100]
@ul[spaced]
- SQL statements are issued from a gateway node.
- The gateway node locates node with the leaseholder replica.
- All read and write requests go through the leaseholder, so it contains the latest, verified replica.
- For reads, the leaseholder simply sends back the requested data to the gateway node.
@ulend
@snapend

---
## Reading & Writing
### Example cluster: 3 nodes, 3 tables, 3 ranges, 3 replicas

@snap[midpoint span-100]
![](assets/img/readwrite.png)
@snapend

---

## Reading & Writing
#### Read Scenario 1: Gateway different from leaseholder

@snap[midpoint text-05 span-100]
![](assets/img/read.png)

@snapend

@snap[south span-100]
@ul[spaced]
- Query Table 3 from Node 2 (the gateway node).
- The replica of Range 3 is on Node 3.
- Gateway node sends request to leaseholder node.
- Leaseholder node sends read response to gateway node.
@ulend
@snapend


---

## Reading & Writing
#### Read Scenario 2: Gateway same as leaseholder

@snap[midpoint span-100]
![](assets/img/read2.png)
@snapend

@snap[south span-100]
@ul[spaced]
- Query Table 3 from Node 3 (the gateway and leaseholder node).
@ulend
@snapend

---

## Reading & Writing
### How do writes work in CockroachDB?

@snap[midpoint span-100]
@ul[spaced]
- SQL statements are issued from a gateway node.
- The gateway node locates and sends request to the node with the leaseholder replica.
- Writes are a little more complicated than reads... the leaseholder needs to coordinate with the Raft leader.
@ulend
@snapend

---

## Reading & Writing
#### Write Scenario 1: Gateway node different from leaseholder/leader

@snap[midpoint span-55]
![](assets/img/write.png)
@snapend

@snap[south span-100]
@ul[spaced text-10]
- Issue write from Node 3 to Table 1.
- Leaseholder node for Range 1 is the same as the Raft leader node (Node 1).
- Write request sent to Node 1.
@ulend
@snapend

---

## Reading & Writing
#### Write Scenario 1: Gateway node different from leaseholder/leader

@snap[midpoint span-55]
![](assets/img/write.png)
@snapend

@snap[south span-100]
@ul[spaced text-10]
- Node 1 writes to Raft log and sends write request to follower nodes to append to Raft logs.
- The first follower node to receive and append write to Raft log sends acknowledgement response to the Raft leader.
- The Raft leader notifies the gateway node of successful write.
@ulend
@snapend

---

## Reading & Writing
#### Write Scenario 2: Gateway node same as leaseholder/leader

@snap[midpoint span-55]
![](assets/img/write2.png)
@snapend

@snap[south span-100]
@ul[spaced]
- Issue write from Node 1 to Table 1 (the gateway, leaseholder, and Raft leader node).
- Node 1 writes to Raft log and sends write request to follower nodes to append to Raft logs.
@ulend
@snapend

---

## Reading & Writing
#### Write Scenario 2: Gateway node same as leaseholder/leader

@snap[midpoint span-55]
![](assets/img/write2.png)
@snapend

@snap[south span-100]
@ul[spaced]
- The first follower node to receive and append write to Raft log sends acknowledgement response to the Raft leader.
@ulend
@snapend

---

## Agenda

@snap[midpoint text-left span-100]
@ul[spaced text-25]
- What is CockroachDB?
- How is data stored?
- How is data replicated and distributed?
- How do reads and writes behave in CockroachDB?
- **How does CockroachDB tolerate failures?**
- How does CockroachDB guarantee consistency?
@ulend
@snapend

---

## Fault-tolerance
### How does CockroachDB tolerate failures?

@snap[midpoint span-100]
@ul[spaced]
- CockroachDB tolerates failures with replication and automated repair.
- CockroachDB can survive **(*n* - 1)/2** failures, where *n* is the replication factor of a piece of data. (e.g. A 5x replication can survive **(5-1)/2 = 2** failures.)
@ulend
@snapend

---

## Failures
### Scenario 1: Single node fails on 3-node cluster

@snap[midpoint text-05 span-100]
![](assets/img/fault.png)
@snapend

@snap[south span-100]
@ul[spaced]
- Majority (2/3) still possible for consensus.
- Cluster remains available.
@ulend
@snapend

---

## Failures
### Scenario 2: Two nodes fail on 3-node cluster

@snap[midpoint text-05 span-100]
![](assets/img/fault2.png)
@snapend

@snap[south span-100]
@ul[spaced]
- Majority not possible for consensus.
- Cluster unavailable.
@ulend
@snapend

---

## Automated Repair
### Scenario 3: One node fails in 4-node cluster

@snap[midpoint text-05 span-100]
![](assets/img/repair.png)
@snapend

@snap[south span-100]
@ul[spaced]
- Ranges are under-replicated.
- Cluster waits for Node 2 to come back online.
@ulend
@snapend

---

## Automated Repair
### Scenario 3: One node fails in 4-node cluster

@snap[midpoint text-05 span-100]
![](assets/img/repair2.png)
@snapend

@snap[south span-100]
@ul[spaced]
- After 5 minutes, if the node doesn't come back online, the cluster automatically rebalances.
@ulend
@snapend

---

## Agenda

@snap[midpoint text-left span-100]
@ul[spaced text-25]
- What is CockroachDB?
- How is data stored?
- How is data replicated and distributed?
- How do reads and writes behave in CockroachDB?
- How does CockroachDB tolerate failures?
- **How does CockroachDB guarantee consistency?**
@ulend
@snapend

---

## Consistency
### How does CockroachDB guarantee consistency?

@snap[midpoint span-100]
@ul[spaced]
- Recall that transactions in CRDB guarantee that data operations are atomic, strongly serializable ("isolated"), and durable (the **A**, **I**, and **D** in ACID).
- **C** is the consistency...
@ulend
@snapend

---

## Consistency
### How does CockroachDB guarantee consistency?

@snap[midpoint span-100]
@ul[spaced]
- Replicas of data remain consistent across the distributed database, despite concurrent requests (i.e. no "stale reads").
- CRDB guarantees consistent reads with multi-version concurrency control (MVCC). New writes do not overwrite old values. Instead, they create a new version with a later timestamp.
- CRDB guarantees consistent writes with Raft.
@ulend
@snapend

---

# Questions?
