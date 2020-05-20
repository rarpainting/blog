# 不同 DBMS 之间的差异

## MySQL

[Differences Between the NDB and InnoDB Storage Engines](https://dev.mysql.com/doc/refman/5.7/en/mysql-cluster-ndb-innodb-engines.html)

- Paxos

### InnoDB Cluster

- official HA cluster

### NDB Cluster

- sharding + HA cluster
- 没有 MVCC , 没有 Data Compression (??)
- Transaction: **READ COMMITTED / RC**
- 大行/Large Row(>14K) 支持: BLOB, TEXT
- 只支持异步复制, 没有组复制(group replication)

## MongoDB

[MongoDB and Raft](http://henrikingo.github.io/presentations/PGDay%20Russia%202017%20-%20MongoDB%20and%20Raft/index.html#/pv0)

- cluster: 异步 Raft

## Redis

- Gossip

## TiDB

- PD: Raft
