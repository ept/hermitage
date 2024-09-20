Testing TiDB transaction isolation levels
==========================================

This file is a subset of mysql.md.

These tests were run with TiDB 7.5.0. Use tiup to set up the TiDB cluster.
See http://tiup.io for installation instructions. Cluster was started using
the the command line:

   tiup playground v7.5.0 --db 2 --pd 3 --kv 3

TiDB only supports READ-COMMITTED and SNAPSHOT-ISOLATION isolation levels. TiDB
snapshot isolation differs from the ANSI SQL REPEATABLE-READ.
See, https://docs-archive.pingcap.com/tidb/v7.2/transaction-isolation-levels#difference-between-tidb-and-ansi-repeatable-read

TiDB maps SNAPSHOT-ISOLATION to REPEATABLE-READ. See above link for differences with InnoDB REPEATABLE-READ.

The above tiup command line sets up two SQL nodes "--db 2". Since TiDB is a
compute and storage disaggregated distributed you cn use the MySQL client
to connect to different nodes too.

Setup (before every test case):

```sql
create table test (id int primary key, value int) engine=innodb;
insert into test (id, value) values (1, 10), (2, 20);
```

To see the current isolation level:

```sql
select @@tx_isolation;
```

Read Committed basic requirements (G0, G1a, G1b, G1c)
-----------------------------------------------------

TiDB "read committed" prevents Aborted Reads (G1a):

```sql
set session transaction isolation level read committed; begin; -- T1
set session transaction isolation level read committed; begin; -- T2
update test set value = 101 where id = 1; -- T1
select * from test; -- T2. Still shows 1 => 10
rollback; -- T1
select * from test; -- T2. Still shows 1 => 10
update test set value = 101 where id = 1;commit; -- T2
```

TiDB "read committed" prevents Intermediate Reads (G1b):

```sql
set session transaction isolation level read committed; begin; -- T1
set session transaction isolation level read committed; begin; -- T2
update test set value = 101 where id = 1; -- T1
select * from test; -- T2. Still shows 1 => 10
update test set value = 11 where id = 1; -- T1
commit; -- T1
select * from test; -- T2. Now shows 1 => 11
commit; -- T2
```

TiDB "read committed" prevents Circular Information Flow (G1c):

```sql
set session transaction isolation level read committed; begin; -- T1
set session transaction isolation level read committed; begin; -- T2
update test set value = 11 where id = 1; -- T1
update test set value = 22 where id = 2; -- T2
select * from test where id = 2; -- T1. Still shows 2 => 20
select * from test where id = 1; -- T2. Still shows 1 => 10
commit; -- T1
commit; -- T2
```


Observed Transaction Vanishes (OTV)
-----------------------------------

TiDB "read committed" prevents Observed Transaction Vanishes (OTV):

```sql
set session transaction isolation level read committed; begin; -- T1
set session transaction isolation level read committed; begin; -- T2
set session transaction isolation level read committed; begin; -- T3
update test set value = 11 where id = 1; -- T1
update test set value = 19 where id = 2; -- T1
update test set value = 12 where id = 1; -- T2. BLOCKS
commit; -- T1. This unblocks T2
select * from test; -- T3. Shows 1 => 11, 2 => 19
update test set value = 18 where id = 2; -- T2
select * from test; -- T3. Shows 1 => 11, 2 => 19
commit; -- T2
select * from test; -- T3. Shows 1 => 12, 2 => 18
commit; -- T3
```


Predicate-Many-Preceders (PMP)
------------------------------

TiDB "read committed" does not prevent Predicate-Many-Preceders (PMP):

```sql
set session transaction isolation level read committed; begin; -- T1
set session transaction isolation level read committed; begin; -- T2
select * from test where value = 30; -- T1. Returns nothing
insert into test (id, value) values(3, 30); -- T2
commit; -- T2
select * from test where value % 3 = 0; -- T1. Returns the newly inserted row
commit; -- T1
```

TiDB "repeatable read" prevents Predicate-Many-Preceders (PMP) for read predicates:

```sql
set session transaction isolation level repeatable read; begin; -- T1
set session transaction isolation level repeatable read; begin; -- T2
select * from test where value = 30; -- T1. Returns nothing
insert into test (id, value) values(3, 30); -- T2
commit; -- T2
select * from test where value % 3 = 0; -- T1. Still returns nothing
commit; -- T1
```

TiDB "read committed" does not prevent Predicate-Many-Preceders (PMP) for write predicates -- example from Postgres documentation:

```sql
set session transaction isolation level read committed; begin; -- T1
set session transaction isolation level read committed; begin; -- T2
update test set value = value + 10; -- T1
select * from test; -- T2. Returns 1 => 10, 2 => 20
delete from test where value = 20;  -- T2, BLOCKS
commit; -- T1. This unblocks T2
select * from test; -- T2. Returns 2 => 30
commit; -- T2
```

TiDB "repeatable read" does not prevent Predicate-Many-Preceders (PMP) for write predicates -- example from Postgres documentation:

```sql
set session transaction isolation level repeatable read; begin; -- T1
set session transaction isolation level repeatable read; begin; -- T2
update test set value = value + 10; -- T1
select * from test where value = 20; -- T2. Returns 2 => 20
delete from test where value = 20;  -- T2, BLOCKS
commit; -- T1. This unblocks T2
select * from test; -- T2. Returns 2 => 20, despite rows with value 20 ostensibly being deleted
commit; -- T2
```

Lost Update (P4)
----------------

TiDB "repeatable read" does not prevent Lost Update (P4):

```sql
set session transaction isolation level repeatable read; begin; -- T1
set session transaction isolation level repeatable read; begin; -- T2
select * from test where id = 1; -- T1
select * from test where id = 1; -- T2
update test set value = 11 where id = 1; -- T1
update test set value = 11 where id = 1; -- T2, BLOCKS
commit; -- T1
commit; -- T2
```


Read Skew (G-single)
--------------------

TiDB "read committed" does not prevent Read Skew (G-single):

```sql
set session transaction isolation level read committed; begin; -- T1
set session transaction isolation level read committed; begin; -- T2
select * from test where id = 1; -- T1. Shows 1 => 10
select * from test where id = 1; -- T2
select * from test where id = 2; -- T2
update test set value = 12 where id = 1; -- T2
update test set value = 18 where id = 2; -- T2
commit; -- T2
select * from test where id = 2; -- T1. Shows 2 => 18
commit; -- T1
```

TiDB "repeatable read" prevents Read Skew (G-single) on a read-only transaction:

```sql
set session transaction isolation level repeatable read; begin; -- T1
set session transaction isolation level repeatable read; begin; -- T2
select * from test where id = 1; -- T1. Shows 1 => 10
select * from test where id = 1; -- T2
select * from test where id = 2; -- T2
update test set value = 12 where id = 1; -- T2
update test set value = 18 where id = 2; -- T2
commit; -- T2
select * from test where id = 2; -- T1. Shows 2 => 20
commit; -- T1
```

TiDB "repeatable read" prevents Read Skew (G-single) -- test using predicate dependencies:

```sql
set session transaction isolation level repeatable read; begin; -- T1
set session transaction isolation level repeatable read; begin; -- T2
select * from test where value % 5 = 0; -- T1
update test set value = 12 where value = 10; -- T2
commit; -- T2
select * from test where value % 3 = 0; -- T1. Returns nothing
commit; -- T1
```

TiDB "repeatable read" does not prevent Read Skew (G-single) on a write predicate:

```sql
set session transaction isolation level repeatable read; begin; -- T1
set session transaction isolation level repeatable read; begin; -- T2
select * from test where id = 1; -- T1. Shows 1 => 10
select * from test; -- T2
update test set value = 12 where id = 1; -- T2
update test set value = 18 where id = 2; -- T2
commit; -- T2
delete from test where value = 20; -- T1. Doesn't delete anything
select * from test where id = 2;   -- T1. Shows 2 => 20
commit; -- T1
```

Write Skew (G2-item)
--------------------

TiDB "repeatable read" does not prevent Write Skew (G2-item):

```sql
set session transaction isolation level repeatable read; begin; -- T1
set session transaction isolation level repeatable read; begin; -- T2
select * from test where id in (1,2); -- T1
select * from test where id in (1,2); -- T2
update test set value = 11 where id = 1; -- T1
update test set value = 21 where id = 2; -- T2
commit; -- T1
commit; -- T2
```

Anti-Dependency Cycles (G2)
---------------------------

TiDB "repeatable read" does not prevent Anti-Dependency Cycles (G2):

```sql
set session transaction isolation level repeatable read; begin; -- T1
set session transaction isolation level repeatable read; begin; -- T2
select * from test where value % 3 = 0; -- T1
select * from test where value % 3 = 0; -- T2
insert into test (id, value) values(3, 30); -- T1
insert into test (id, value) values(4, 42); -- T2
commit; -- T1
commit; -- T2
select * from test where value % 3 = 0; -- Either. Returns 3 => 30, 4 => 42
```
