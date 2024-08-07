Testing Memgraph's transaction isolation levels
===============================================


### You can find all Memgraph tests [here](https://github.com/memgraph/memgraph/blob/master/tests/manual/test_isolation_level.py).

[Memgraph](https://memgraph.com/) is a graph database which uses Cypher query language in its core. Memgraph currently supports three isolation levels, from the highest to the lowest:

**SNAPSHOT_ISOLATION (default)** - guarantees that all reads made in a transaction will see a consistent snapshot of the database, 
and the transaction itself will successfully commit only if no updates it has made conflict with any concurrent updates made 
since that snapshot. Protects users from observing Dirty Read, Non-repeatable Read and Phantom phenomena as described in 
ANSI/ISO SQL-92 standard.

**READ_COMMITTED** - guarantees that any data read was committed at the moment it is read. It protects users from observing Dirty Read phenomenon as described in ANSI/ISO SQL-92 standard.

**READ_UNCOMMITTED** - one transaction may read not yet committed changes made by other transactions. 
Doesn't protect users from any of the three phenomena described in ANSI/ISO SQL-92 standard. In order to not mess up data consistency, this isolation level should only be used in the read-only access mode.

| Phenomenon          | Description                                                                                                                                          | Disallowed by                                              |
| ------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------- |
| Dirty Read          | Transaction reads an object that was never committed by other transaction.                                                                           | SNAPSHOT ISOLATION, READ COMMITTED                         |
| Non-repeatable Read | Transaction reads an object twice. When the txn reads the object 2nd time, it receives the modified value because other txn modified it.             | SNAPSHOT ISOLATION                                         |
| Phantom             | Transaction reads objects meeting a certain condition and then finds additional objects when reading 2nd time because another txn added new objects. | SNAPHSOT ISOLATION                                         |


Default SNAPSHOT ISOLATION level implicitly supports isolation levels (as specified in [Adya's thesis](https://pmg.csail.mit.edu/papers/adya-phd.pdf)): **PL-1, PL-MSR (Monotonic Snapshot Reads), PL-2, PL-2', PL-2'', PL-2L, PL-CS (Cursor Stability) and PL-2+ (consistent view)**.

Too see how to start Memgraph, please refer to our [docs](https://memgraph.com/docs/getting-started/install-memgraph). You can change the
global isolation level using `--isolation-level` flag or you can use Cypher query language to change isolation level in various scopes (GLOBAL, SESSION, TRANSACTION).

**Contributors**:
- [**spacejam**](https://github.com/spacejam)
- [**as51340**](https://github.com/as51340)
