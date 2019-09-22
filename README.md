# toyDB

[![Build Status](https://cloud.drone.io/api/badges/erikgrinaker/toydb/status.svg)](https://cloud.drone.io/erikgrinaker/toydb)

Distributed SQL database in Rust, written as a learning project.

The primary goal is to build a minimally functional yet correct distributed database. Performance, security, reliability, and convenience are non-goals.

## Usage

A local five-node cluster can be started on `localhost` ports `9601` to `9605` by running:

```
$ (cd sandbox && docker-compose up --build)
```

A command-line REPL client can be built and used with the node on `localhost` port `9605`
by running:

```
$ cargo run --bin toysql
Connected to node "toydb-e" (version 0.1.0). Enter !help for instructions.
toydb>
```

A basic subset of SQL has been partially implemented, e.g.:

```
toydb> CREATE TABLE movies (id INTEGER PRIMARY KEY, title VARCHAR NOT NULL)
toydb> INSERT INTO movies VALUES (1, 'Sicario'), (2, 'Stalker'), (3, 'Her')
toydb> SELECT * FROM movies
1|Sicario
2|Stalker
3|Her
```

## Project Outline

- [x] **Networking:** gRPC for internal and external communication, no security.

- [x] **Client:** Simple interactive REPL client over gRPC.

- [x] **Consensus:** Self-written Raft implementation with linearizable replication of arbitrary state machines.

- [ ] **Storage:** Self-written key-value store using B+trees and possibly LSM-trees, with MessagePack for serialization.

- [x] **Data Types:** Support for nulls, booleans, 64-bit integers, 64-bit floats, and UTF-8 strings up to 1 KB.

- [ ] **Schemas:** Compulsory singluar primary keys, unique and foreign key constraints, indexes.

- [ ] **Transactions:** Self-written ACID-compliant transaction engine with MVCC-based serializable snapshot isolation.

- [x] **Query Engine:** Self-written iterator-based engine with simple heuristic optimizer.

- [ ] **Language:** Self-written SQL parser with support for:

  - [x] `[CREATE|DROP] TABLE ...`
  - [ ] `[CREATE|DROP] INDEX ...`
  - [ ] `BEGIN`, `COMMIT`, and `ROLLBACK`
  - [x] `INSERT INTO ... (...) VALUES (...)`
  - [x] `UPDATE ... SET ... WHERE ...`
  - [x] `DELETE FROM ... WHERE ...`
  - [x] `SELECT ... FROM ... WHERE ... ORDER BY ... LIMIT ... OFFSET ...`
    - [ ] `GROUP BY ... HAVING ...`
    - [ ] `[INNER|LEFT|RIGHT|FULL|CROSS] JOIN`
  - [ ] `EXPLAIN SELECT ...`

- [ ] **Verification:** [Jepsen](https://github.com/jepsen-io/jepsen) test suite.

## Known Issues

Below is an incomplete list of known issues preventing this from being a "real" database.

### Networking

* **No security:** all network traffic is unauthenticated and in plaintext; any request from any source is accepted.

### Raft

* **Cluster reconfiguration:** the Raft cluster must consist of a static set of nodes available via static IP addresses. It is not possible to resize the cluster without a full cluster restart.

* **Single node processing:** all operations (both reads and writes) are processed by a single Raft thread on a single node (the master), and the system consists of a single Raft cluster, preventing horizontal scalability and efficient resource utilization.

* **Client call retries:** there is currently no retries of client-submitted operations, and if a node processing or proxying an operation changes role then the call is dropped.

* **State machine errors:** errors during state machine mutations currently crash the node - it may be beneficial to support user errors which simply skip the erroring log entry.

* **Log replication optimization:** currently only the simplest version of the Raft log replication protocol is implemented, without snapshots or rapid log replay (i.e. replication of old log entries is retried one by one until a common base entry is found).

### Schema

* **Single database:** only a single, unnamed database is supported per ToyDB cluster.

* **Schema changes:** schema changes other than creating or dropping tables and indexes is not supported, i.e. there is no `ALTER TABLE`.

### Query Engine

* **Type checking:** query type checking (e.g. `SELECT a + b` must receive two numbers) is done at query evaluation time, not at query compile time.

* **Ordering:** it's only possible to order on output columns of the `SELECT` statement.
