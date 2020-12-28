Testing CockroachDB transaction isolation levels
===============================================

CockroachDB executes all transactions at the strongest transaction isolation level: **Serializable**.


These tests were run with CockroachDB 20.2.3.

Setup (before every test case):

```sql
create table test (id int primary key, value int);
insert into test (id, value) values (1, 10), (2, 20);
```

Basic requirements (G0, G1a, G1b, G1c)
-----------------------------------------------------

CockroachDB "serializable" prevents Write Cycles (G0) by locking updated rows:

```sql
begin; -- T1
begin; -- T2
update test set value = 11 where id = 1; -- T1
update test set value = 12 where id = 1; -- T2, BLOCKS
update test set value = 21 where id = 2; -- T1
select * from test; -- T1. Shows 1 => 11, 2 => 21
commit; -- T1. This unblocks T2
update test set value = 22 where id = 2; -- T2
commit; -- T2
select * from test; -- either. Shows 1 => 12, 2 => 22
```

CockroachDB "serializable" prevents Aborted Reads (G1a):

```sql
begin; -- T1
begin; -- T2
update test set value = 101 where id = 1; -- T1
select * from test; -- T2. Still shows 1 => 10
abort;  -- T1
select * from test; -- T2. Still shows 1 => 10
commit; -- T2
```

CockroachDB "serializable" prevents Intermediate Reads (G1b):

```sql
begin; -- T1
begin; -- T2
update test set value = 101 where id = 1; -- T1
select * from test; -- T2. Still shows 1 => 10
update test set value = 11 where id = 1; -- T1
commit; -- T1
select * from test; -- T2. Still shows 1 => 10
commit; -- T2
```

CockroachDB "serializable" prevents Circular Information Flow (G1c):

```sql
begin; -- T1
begin; -- T2
update test set value = 11 where id = 1; -- T1
update test set value = 22 where id = 2; -- T2
select * from test where id = 2; -- T1. Still shows 2 => 20
select * from test where id = 1; -- T2. Still shows 1 => 10
commit; -- T1
commit; -- T2, ERROR: restart transaction: TransactionRetryWithProtoRefreshError: TransactionRetryError: retry txn (RETRY_SERIALIZABLE - failed preemptive refresh)
```


Observed Transaction Vanishes (OTV)
-----------------------------------

CockroachDB "serializable" prevents Observed Transaction Vanishes (OTV):

```sql
begin; -- T1
begin; -- T2
begin; -- T3
update test set value = 11 where id = 1; -- T1
update test set value = 19 where id = 2; -- T1
update test set value = 12 where id = 1; -- T2. BLOCKS
commit; -- T1. This unblocks T2
select * from test where id = 1; -- T3. Shows 1 => 10
update test set value = 18 where id = 2; -- T2
select * from test where id = 2; -- T3. Shows 2 => 20
commit; -- T2
select * from test where id = 2; -- T3. Shows 2 => 20
select * from test where id = 1; -- T3. Shows 1 => 10
commit; -- T3
```


Predicate-Many-Preceders (PMP)
------------------------------

CockroachDB "serializable" prevents Predicate-Many-Preceders (PMP):

```sql
begin; -- T1
begin; -- T2
select * from test where value = 30; -- T1. Returns nothing
insert into test (id, value) values(3, 30); -- T2
commit; -- T2
select * from test where value % 3 = 0; -- T1. Still returns nothing
commit; -- T1
```

CockroachDB "serializable" prevents Predicate-Many-Preceders (PMP) for write predicates -- example from Postgres documentation:

```sql
begin; -- T1
begin; -- T2
update test set value = value + 10 where true; -- T1
delete from test where value = 20;  -- T2, BLOCKS
commit; -- T1. T2 now prints out "ERROR: restart transaction: TransactionRetryWithProtoRefreshError: TransactionRetryError: retry txn (RETRY_WRITE_TOO_OLD - WriteTooOld flag converted to WriteTooOldError)"
abort;  -- T2. There's nothing else we can do, this transaction has failed
```


Lost Update (P4)
----------------

CockroachDB "serializable" prevents Lost Update (P4):

```sql
begin; -- T1
begin; -- T2
select * from test where id = 1; -- T1
select * from test where id = 1; -- T2
update test set value = 11 where id = 1; -- T1
update test set value = 11 where id = 1; -- T2, BLOCKS
commit; -- T1. T2 now prints out "ERROR: restart transaction: TransactionRetryWithProtoRefreshError: WriteTooOldError: write at timestamp 1609143635.427526584,0 too old; wrote at 1609143665.238193548,3"
abort;  -- T2. There's nothing else we can do, this transaction has failed
```


Read Skew (G-single)
--------------------

CockroachDB "serializable" prevents Read Skew (G-single):

```sql
begin; -- T1
begin; -- T2
select * from test where id = 1; -- T1. Shows 1 => 10
select * from test where id = 1; -- T2
select * from test where id = 2; -- T2
update test set value = 12 where id = 1; -- T2
update test set value = 18 where id = 2; -- T2
commit; -- T2
select * from test where id = 2; -- T1. Shows 2 => 20
commit; -- T1
```

CockroachDB "serializable" prevents Read Skew (G-single) -- test using predicate dependencies:

```sql
begin; -- T1
begin; -- T2
select * from test where value % 5 = 0; -- T1
update test set value = 12 where value = 10; -- T2
commit; -- T2
select * from test where value % 3 = 0; -- T1. Returns nothing
commit; -- T1
```

CockroachDB "serializable" prevents Read Skew (G-single) -- test using write predicate:

```sql
begin; -- T1
begin;-- T2
select * from test where id = 1; -- T1. Shows 1 => 10
select * from test; -- T2
update test set value = 12 where id = 1; -- T2
update test set value = 18 where id = 2; -- T2
commit; -- T2
delete from test where value = 20; -- T1. Prints "ERROR: restart transaction: TransactionRetryWithProtoRefreshError: TransactionRetryError: retry txn (RETRY_WRITE_TOO_OLD - WriteTooOld flag converted to WriteTooOldError)"
abort; -- T1. There's nothing else we can do, this transaction has failed
```


Write Skew (G2-item)
--------------------

CockroachDB "serializable" prevents Write Skew (G2-item):

```sql
begin; -- T1
begin; -- T2
select * from test where id in (1,2); -- T1
select * from test where id in (1,2); -- T2
update test set value = 11 where id = 1; -- T1
update test set value = 21 where id = 2; -- T2
commit; -- T1
commit; -- T2. Prints out "ERROR: restart transaction: TransactionRetryWithProtoRefreshError: TransactionRetryError: retry txn (RETRY_SERIALIZABLE - failed preemptive refresh)"
```


Anti-Dependency Cycles (G2)
---------------------------

CockroachDB "serializable" prevents Anti-Dependency Cycles (G2):

```sql
begin; -- T1
begin; -- T2
select * from test where value % 3 = 0; -- T1
select * from test where value % 3 = 0; -- T2
insert into test (id, value) values(3, 30); -- T1
insert into test (id, value) values(4, 42); -- T2
commit; -- T1
commit; -- T2. Prints out "ERROR: restart transaction: TransactionRetryWithProtoRefreshError: TransactionRetryError: retry txn (RETRY_SERIALIZABLE - failed preemptive refresh)"
```

CockroachDB "serializable" prevents Anti-Dependency Cycles (G2) -- Fekete et al's example with two anti-dependency edges:

```sql
begin; -- T1
select * from test; -- T1. Shows 1 => 10, 2 => 20
begin; -- T2
update test set value = value + 5 where id = 2; -- T2
commit; -- T2
begin; -- T3
select * from test; -- T3. Shows 1 => 10, 2 => 25
commit; -- T3
update test set value = 0 where id = 1; -- T1. 
commit; -- T1. Prints out "ERROR: restart transaction: TransactionRetryWithProtoRefreshError: TransactionRetryError: retry txn (RETRY_SERIALIZABLE - failed preemptive refresh)"
```
