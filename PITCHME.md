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
- A SQL client.
- A key-value store.
- Other components that handle distributing, replicating, and storing data in a transactional way that guarantees **ACID** properties.
@snapend

---

## What is CockroachDB?
### ACID Guarantee

@snap[midpoint span-100]
@ul[spaced]
- **A**tomic (*Transactions happen or they don't.*)
- **C**onsistent (*Data is always in a valid state, across all locations.*)
- **I**solated (*Transactions are separate, and strongly serializable.*)
- **D**urable (*The data persists, even with failures.*)
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

## Storage
### How is data stored in CockroachDB?

@snap[west span-100]
@ul[spaced]
- **SQL interface**: Users access data in CockroachDB as entries in rows and columns of a table, with SQL statements.
- **Key-value store**: Under the hood, data are stored in partitions ("ranges") of key-value pairs in a sorted key-value store.
@ulend
@snapend

---

## Storage
### SQL interface

@snap[midpoint span-100]
@ul[spaced]
- The user accesses the database of tables through the CockroachDB SQL interface.
- Primary key values (rows in a primary key column) uniquely identify rows of data.
- From the SQL perspective, data is represented as being in one place.
@ulend
@snapend

---

## Storage
### Key-value store

@snap[midpoint span-100]
@ul[spaced]
- CockroachDB stores all data, including table data, metadata, and indexes, as key-value pairs in a key-value store powered by RocksDB.
- Each key in the key-value store is a unique ID based on the table ID, the primary key column row value, and the column ID.
- Each value in the key-value store is the value of the data entry for the corresponding unique key.
@ulend
@snapend

---

## Storage
### Key-value store

@snap[midpoint text-right span-100]
![](assets/img/keyspace1.png)
@snapend

@snap[midpoint text-left span-80]
The "keyspace" can be represented as a monolothic, sorted map.
@snapend

---

## Storage
### Key-value store

@snap[midpoint span-100]
![](assets/img/keyspace2.png)
@snapend

@snap[]
The keyspace is partitioned into ranges.
@snapend

---

## Storage
### Key-value store

@snap[midpoint span-100]
![](assets/img/keyspace3.png)

These ranges are replicated and distributed to nodes.
@snapend

---

## Replication & Distribution
### How is data replicated and distributed in CockroachDB?

@snap[midpoint span-100]
@ul[spaced]
- A **node** is an instance of CockroachDB.
- A **cluster** is a group of nodes. The nodes in a cluster communicate through requests and responses on a gossip network.
- Data in the key-value store is partitioned into **ranges** of up to 64 MiB.
@ulend
@snapend

---

## Replication & Distribution
### How is data replicated and distributed in CockroachDB?

@snap[midpoint span-100]
@ul[spaced]
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
- The **gateway node** is the node through which a user accesses the SQL interface.
- One of the replicas holds a "range lease". This replica manages the read and write requests for its range. This replica is known as the **leaseholder**, and its node is the **leaseholder node**.
- When a user submits a SQL statement, the gateway node identifies the leaseholder for the range of interest, and sends the read or write request to the leaseholder. (More on reads and writes  later...)
@ulend
@snapend

---

## Replication & Distribution
### Consensus with Raft

@snap[west span-50]
@ul[spaced]
- CockroachDB uses the Raft consensus algorithm to determine which replica to distribute across a cluster.
- The replica chosen is the **leader**. The other replicas are the **followers**.
- Timeouts run on each node to determine when to hold new elections.
@ulend
@snapend

@snap[east span-50]
![](assets/img/raft.png)
[Ongaro and Ousterhout (2014)](https://www.usenix.org/system/files/conference/atc14/atc14-paper-ongaro.pdf)
@snapend

---

## Reading & Writing
### How do reads work in CockroachDB?

@snap[midpoint span-100]
@ul[spaced]
- SQL statements are issued from a gateway node.
- The gateway node locates and sends request to the node with the leaseholder replica.
- Since all read and write requests go through the leaseholder, it contains the latest, verified replica. The leaseholder simply sends back the requested data to the gateway node.
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
### Read Scenario 1: Gateway different from leaseholder

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
### Read Scenario 2: Gateway same as leaseholder

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
### Write Scenario 1: Gateway node different from leaseholder and Raft leader

@snap[midpoint span-70]
![](assets/img/write.png)
@snapend

@snap[south span-100]
@ul[spaced text-10]
- Issue write to Table 1 from Node 3 (the gateway node).
- Leaseholder node for Range 1 is the same as the Raft leader node (Node 1).
- Write request sent to Node 1.
@ulend
@snapend

---

## Reading & Writing
### Write Scenario 1: Gateway node different from leaseholder and Raft leader

@snap[midpoint span-70]
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
### Write Scenario 2: Gateway node same as leaseholder and Raft leader

@snap[midpoint span-70]
![](assets/img/write2.png)
@snapend

@snap[south span-100]
@ul[spaced]
- Issue write to Table 1 from Node 1 (the gateway, leaseholder, and Raft leader node).
- Node 1 writes to Raft log and sends write request to follower nodes to append to Raft logs.
- The first follower node to receive and append write to Raft log sends acknowledgement response to the Raft leader.
@ulend
@snapend

---

## Fault-tolerance
### How does CockroachDB tolerate failures?

@snap[midpoint span-100]
@ul[spaced]
- CockroachDB tolerates failures with replication and automated repair.
- CockroachDB can survive (*n* - 1)/2 failures, where *n* is the replication factor of a piece of data. (e.g. A 5x replication can survive (5-1)/2 = 2 failures.)
@ulend
@snapend

---

## Failures
### Scenario 1: Single node fails on 3-node cluster

@snap[midpoint text-05 span-100]
![](assets/img/fault.png)
(*stolen directly from training slide*)
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
(*stolen directly from training slide*)
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
(*stolen directly from training slide*)
@snapend

@snap[south span-100]
@ul[spaced]
- Nodes are under-replicated.
- Cluster waits for Node 2 to come back online.
@ulend
@snapend

---

## Automated Repair
### Scenario 3: One node fails in 4-node cluster

@snap[midpoint text-05 span-100]
![](assets/img/repair2.png)
(*stolen directly from training slide*)
@snapend

@snap[south span-100]
@ul[spaced]
- After 5 minutes, if the node doesn't come back online, the cluster automatically rebalances.
@ulend
@snapend

---

## Consistency
### How does CockroachDB guarantee consistency?

@snap[midpoint span-100]
@ul[spaced]
- Recall that transactions are atomic, serializable ("isolated"), and durable (the A, I, and D in ACID).
- C is the consistency.
@ulend
@snapend

---

## Consistency

@snap[midpoint span-100]
@ul[spaced]
- Recall that transactions in CRDB guarantee that data operations are atomic, strongly serializable ("isolated"), and durable (the **A**, **I**, and **D** in ACID).
- **C** is the consistency...
- Replica of data remain consistent across the distributed database, despite concurrent requests. I.e. "no stale reads."
- CRDB guarantees consistent reads with multi-version concurrency control (MVCC).
- CRDB guarantees consistent writes with Raft.
@ulend
@snapend

---
