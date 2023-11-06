# ReadDB
I'm interested in database systems and I plan to learn this field broadly by reading papers, so I make a repo as a notebook as I read papers. 

## Abbreviation
Transaction - Tx  
Concurrency Control - Cc
Repeatable Read - RR
Snapshot-Isolation - SI

## List
* TiDB

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

TiDB uses CBO to estimate cost of queries based on size of tuple/column/index and number of tuple/region. It uses these metrics to build equations to compute cost and select the min cost plan, which may go to column store, row store or both, based on which are the targets(tuple/column/index) to scan.

Benchmark: Higher throughputs than CockRoachDB even under pessimistic cc. For OLAP, it's always better to access from both row store and column store than accessing single store.
 
