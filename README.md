# ReadDB
Reading papers about database systems

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

