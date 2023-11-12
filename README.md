# ReadDB
I'm interested in database systems and I plan to learn this field broadly by reading papers, so I make a repo as a notebook as I read papers.  
Since I target at learning things I'm interested in, the notes may omit many details in the papers.

## Abbreviation
Transaction - Tx  
Concurrency Control - Cc  
Repeatable Read - RR  
Snapshot-Isolation - SI  
Highly Available/High Availability - HA

## List
- [x] TiDB
- [x] Bigtable
- [x] Pavlo, A., Curino, C. and Zdonik, S., 2012, May. Skew-aware automatic database partitioning in shared-nothing, parallel OLTP systems. In Proceedings of the 2012 ACM SIGMOD International Conference on Management of Data (pp. 61-72).
- [ ] Dynamo

## TiDB

A Distributed SQL(aka. NewSQL) system supporting HTAP workloads.

The need for HTAP: Building two systems separately for OLTP and OLAP is just more complicated than supporting both in one system. In addition, the freshness of data read by OLAP engine is important.

Two important concepts in HTAP: _Freshness_ (how recent the data read by analytical queries is) and _isolation_ (avoid performance interference between OLTP and OLAP workloads).

Most important innovation: by introducing a _learner_ role in Raft algorithm which replicates logs from leader to generate columnar data for OLAP. The learner doesn't count into the quorum of Raft requests, nor does it participate in the election process.  Learner asynchronously replicates data from leader.

Row store is called TiKV, column store is called TiFlash. TiKV is based on RocksDB. It stores ordered maps(probably we can just say SSTable) and does range paritions. Each partion is managed by a Raft group, each TiKV server is a node(follower or leader) in the Raft group. So if there are 3 TiKV servers and data is split into 3 partitions, logically there will be 9 paritions in total, since each partition is replicated on all 3 servers. Writes go to leader, which propagates log entries to followers and wait for quorum success. It does some optimizations on Raft execution:  
1. Assume log append success and keep sending new entries before receiving success, if fail just resend.
2. Leader parallelizes appending entries to logs(disk I/O) and sending logs to followers(network latency).
3. Applying changes after append success is also handles by another thread.
4. Follower can also respond to reads if its commit idx >= leader's commit.

Column store is called TiFlash. The logs it learns from the leader are in the format of DML transaction operations(like WAL) including insert, update and delete. An optimization is to compress logs based on TX status and overwritten tuples. TiFlash uses an interesting storage model called Delta Tree. It separates normal tuples and incoming deltas into two spaces, and periodically merges deltas with the tuples. Since incoming deltas are unsorted, it builds a B+ Tree index for the deltas to efficiently merge them. It argues that the read performance of Delta Tree is twice faster then LSM Tree used in TiKV. (What I feel is Delta Tree combines the properties of the two classic storage models - B+ and LSM)

Tx processing: Cc protocol is based on MVCC, but it also provides pessimistic control. TiDB prodives SI or RR. Distributed tx in coordinated by 2PC. The difference on implementaion between optimisitic and pessimistic here is when to acquire locks. For the former, locks are acquired after DMLs complete, when data is writing to TiKVs; for the latter locks are acquired as DMLs are executed in local space.

OLAP optmizations: cropping unneeded columns, eliminating projection, pushing down predicates, deriving predicates, constant folding, eliminating “group by” or outer joins, and unnesting subqueries.


## Bigtable

Target: Scalable, HA, Widely applicable, High performance  

Wide-column, update to a single row is atomic even for multiple columns.  

Sorted String Table with range parition, so client can design row keys such that frequently used-together keys are close to each other. A row range is called tablet.  

Architecture is one master server + many tablet servers, master manages distribution of tablets. Bigtable uses a tree-like structure, stores location of tablet metadata at the root, which stores location of tablets, which store the actual data. Client usually doesn't contact master node, instead uses a DNS-like fashion to lookup and cache tablet locations.  

Writes firstly go to logs on GFS, then go to the memtable, then go to SSTable. The metadata mentioned above contains sstable + log of a tablet, so using the metadata can recover a tablet.
Reads go to the merge view of memtable and sstable.  

Details about SSTable are omitted since I've read about it elsewhere. Immutability makes CC very easy.  

Highlights from benchmarking: Writes are faster than random reads since commits are grouped and the append-only nature. Sequential reads are much faster than random reads due to block-level caching. Load-balancing is not good, which hurts scalability.


## Pavlo12

Two problems in distributed SQL systems: number of distributed transactions and skew. The goal is to generate good database design that minimizes both through automatic partitioning.  
Design options include:  
1. Horizontal partitioning: fragmenting a table by some attributes.
2. Table replication: replicating some read-only/heavy tables on all nodes to save coordination, since it's always available locally from any node's point of view.
3. Secondary index: building index for more columns to reduce accesses of more kinds of queries.
4. Stored procedure routing: send query to the node that has most of the data needed by the query.

Use a large-neighborhood search based on cost model which calculates:

1. CoordinationCost, which is an equation that multiplies number of distributed transaction and number of parition accessed
2. Skew factor, (in brief)which computes the access rate(NumUsedByTx/SumNumUsedByTx) for each partition, then adds up the ratio between this rate and ideal rate(1/NumPartition)

 
