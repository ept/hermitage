Testing CockroachDB transaction isolation levels
===============================================

CockroachDB defaults to executing transactions at the strongest transaction
isolation level: **Serializable**. In CockroachDB 23.2, preview support was
added for the **Read Committed** and **Repeatable Read** isolation levels.

These tests were run with CockroachDB 23.2.0-beta.1.

Setup (once after cluster initialization):
```sql
-- enable preview support for weak isolation levels
set cluster setting sql.txn.read_committed_isolation.enabled = true;
set cluster setting sql.txn.snapshot_isolation.enabled = true;
```

Setup (before every test case):

```sql
create table test (id int primary key, value int);
insert into test (id, value) values (1, 10), (2, 20);
```

To see the current isolation level:

```sql
select current_setting('transaction_isolation');
```

Read Committed basic requirements (G0, G1a, G1b, G1c)
-----------------------------------------------------

CockroachDB "read committed" prevents Write Cycles (G0) by locking updated rows:

```sql
begin; set transaction isolation level read committed; -- T1
begin; set transaction isolation level read committed; -- T2
update test set value = 11 where id = 1; -- T1
update test set value = 12 where id = 1; -- T2, BLOCKS
update test set value = 21 where id = 2; -- T1
select * from test; -- T1. Shows 1 => 11, 2 => 21
commit; -- T1. This unblocks T2
update test set value = 22 where id = 2; -- T2
commit; -- T2
select * from test; -- either. Shows 1 => 12, 2 => 22
```

CockroachDB "read committed" prevents Aborted Reads (G1a):

```sql
begin; set transaction isolation level read committed; -- T1
begin; set transaction isolation level read committed; -- T2
update test set value = 101 where id = 1; -- T1
select * from test; -- T2. Still shows 1 => 10
abort;  -- T1
select * from test; -- T2. Still shows 1 => 10
commit; -- T2
```

CockroachDB "read committed" prevents Intermediate Reads (G1b):

```sql
begin; set transaction isolation level read committed; -- T1
begin; set transaction isolation level read committed; -- T2
update test set value = 101 where id = 1; -- T1
select * from test; -- T2. Still shows 1 => 10
update test set value = 11 where id = 1; -- T1
commit; -- T1
select * from test; -- T2. Now shows 1 => 11
commit; -- T2
```

CockroachDB "read committed" prevents Circular Information Flow (G1c):

```sql
begin; set transaction isolation level read committed; -- T1
begin; set transaction isolation level read committed; -- T2
update test set value = 11 where id = 1; -- T1
update test set value = 22 where id = 2; -- T2
select * from test where id = 2; -- T1. Still shows 2 => 20
select * from test where id = 1; -- T2. Still shows 1 => 10
commit; -- T1
commit; -- T2
```


Observed Transaction Vanishes (OTV)
-----------------------------------

CockroachDB "read committed" prevents Observed Transaction Vanishes (OTV):

```sql
begin; set transaction isolation level read committed; -- T1
begin; set transaction isolation level read committed; -- T2
begin; set transaction isolation level read committed; -- T3
update test set value = 11 where id = 1; -- T1
update test set value = 19 where id = 2; -- T1
update test set value = 12 where id = 1; -- T2. BLOCKS
commit; -- T1. This unblocks T2
select * from test where id = 1; -- T3. Shows 1 => 11
update test set value = 18 where id = 2; -- T2
select * from test where id = 2; -- T3. Shows 2 => 19
commit; -- T2
select * from test where id = 2; -- T3. Shows 2 => 18
select * from test where id = 1; -- T3. Shows 1 => 12
commit; -- T3
```


Predicate-Many-Preceders (PMP)
------------------------------

CockroachDB "read committed" does not prevent Predicate-Many-Preceders (PMP):

```sql
begin; set transaction isolation level read committed; -- T1
begin; set transaction isolation level read committed; -- T2
select * from test where value = 30; -- T1. Returns nothing
insert into test (id, value) values(3, 30); -- T2
commit; -- T2
select * from test where value % 3 = 0; -- T1. Returns the newly inserted row
commit; -- T1
```

CockroachDB "repeatable read" prevents Predicate-Many-Preceders (PMP):

```sql
begin; set transaction isolation level repeatable read; -- T1
begin; set transaction isolation level repeatable read; -- T2
select * from test where value = 30; -- T1. Returns nothing
insert into test (id, value) values(3, 30); -- T2
commit; -- T2
select * from test where value % 3 = 0; -- T1. Still returns nothing
commit; -- T1
```

CockroachDB "read committed" prevents Predicate-Many-Preceders (PMP) for write predicates (unlike Postgres):

```sql
begin; set transaction isolation level read committed; -- T1
begin; set transaction isolation level read committed; -- T2
update test set value = value + 10 where true; -- T1
delete from test where value = 20;  -- T2, BLOCKS
commit; -- T1. This unblocks T2
select * from test where value = 20; -- T2, returns nothing
commit; -- T2
```

CockroachDB "repeatable read" prevents Predicate-Many-Preceders (PMP) for write predicates:

```sql
begin; set transaction isolation level repeatable read; -- T1
begin; set transaction isolation level repeatable read; -- T2
update test set value = value + 10 where true; -- T1
delete from test where value = 20;  -- T2, BLOCKS
commit; -- T1. T2 now prints out "ERROR: restart transaction: TransactionRetryWithProtoRefreshError: WriteTooOldError"
abort;  -- T2. There's nothing else we can do, this transaction has failed
```


Lost Update (P4)
----------------

CockroachDB "read committed" does not prevent Lost Update (P4):

```sql
begin; set transaction isolation level read committed; -- T1
begin; set transaction isolation level read committed; -- T2
select * from test where id = 1; -- T1
select * from test where id = 1; -- T2
update test set value = 11 where id = 1; -- T1
update test set value = 11 where id = 1; -- T2, BLOCKS
commit; -- T1. This unblocks T2, so T1's update is overwritten
commit; -- T2
```

CockroachDB "repeatable read" prevents Lost Update (P4):

```sql
begin; set transaction isolation level repeatable read; -- T1
begin; set transaction isolation level repeatable read; -- T2
select * from test where id = 1; -- T1
select * from test where id = 1; -- T2
update test set value = 11 where id = 1; -- T1
update test set value = 11 where id = 1; -- T2, BLOCKS
commit; -- T1. T2 now prints out "ERROR: restart transaction: TransactionRetryWithProtoRefreshError: WriteTooOldError"
abort;  -- T2. There's nothing else we can do, this transaction has failed
```


Read Skew (G-single)
--------------------

CockroachDB "read committed" does not prevent Read Skew (G-single):

```sql
begin; set transaction isolation level read committed; -- T1
begin; set transaction isolation level read committed; -- T2
select * from test where id = 1; -- T1. Shows 1 => 10
select * from test where id = 1; -- T2. Shows 1 => 10
select * from test where id = 2; -- T2. Shows 2 => 20
update test set value = 12 where id = 1; -- T2
update test set value = 18 where id = 2; -- T2
commit; -- T2
select * from test where id = 2; -- T1. Shows 2 => 18
commit; -- T1
```

CockroachDB "repeatable read" prevents Read Skew (G-single):

```sql
begin; set transaction isolation level repeatable read; -- T1
begin; set transaction isolation level repeatable read; -- T2
select * from test where id = 1; -- T1. Shows 1 => 10
select * from test where id = 1; -- T2. Shows 1 => 10
select * from test where id = 2; -- T2. Shows 2 => 20
update test set value = 12 where id = 1; -- T2
update test set value = 18 where id = 2; -- T2
commit; -- T2
select * from test where id = 2; -- T1. Shows 2 => 20
commit; -- T1
```

CockroachDB "repeatable read" prevents Read Skew (G-single) -- test using predicate dependencies:

```sql
begin; set transaction isolation level repeatable read; -- T1
begin; set transaction isolation level repeatable read; -- T2
select * from test where value % 5 = 0; -- T1
update test set value = 12 where value = 10; -- T2
commit; -- T2
select * from test where value % 3 = 0; -- T1. Returns nothing
commit; -- T1
```

CockroachDB "repeatable read" prevents Read Skew (G-single) -- test using write predicate:

```sql
begin; set transaction isolation level repeatable read; -- T1
begin; set transaction isolation level repeatable read; -- T2
select * from test where id = 1; -- T1. Shows 1 => 10
select * from test; -- T2
update test set value = 12 where id = 1; -- T2
update test set value = 18 where id = 2; -- T2
commit; -- T2
delete from test where value = 20; -- T1. Prints "ERROR: restart transaction: TransactionRetryWithProtoRefreshError: WriteTooOldError"
abort; -- T1. There's nothing else we can do, this transaction has failed
```


Write Skew (G2-item)
--------------------

CockroachDB "repeatable read" does not prevent Write Skew (G2-item):

```sql
begin; set transaction isolation level repeatable read; -- T1
begin; set transaction isolation level repeatable read; -- T2
select * from test where id in (1,2); -- T1
select * from test where id in (1,2); -- T2
update test set value = 11 where id = 1; -- T1
update test set value = 21 where id = 2; -- T2
commit; -- T1
commit; -- T2
```

CockroachDB "serializable" prevents Write Skew (G2-item):

```sql
begin; set transaction isolation level serializable; -- T1
begin; set transaction isolation level serializable; -- T2
select * from test where id in (1,2); -- T1
select * from test where id in (1,2); -- T2
update test set value = 11 where id = 1; -- T1
update test set value = 21 where id = 2; -- T2
commit; -- T1
commit; -- T2. Prints out "ERROR: restart transaction: TransactionRetryWithProtoRefreshError: TransactionRetryError: retry txn (RETRY_SERIALIZABLE - failed preemptive refresh)"
```


Anti-Dependency Cycles (G2)
---------------------------

CockroachDB "repeatable read" does not prevent Anti-Dependency Cycles (G2):

```sql
begin; set transaction isolation level repeatable read; -- T1
begin; set transaction isolation level repeatable read; -- T2
select * from test where value % 3 = 0; -- T1
select * from test where value % 3 = 0; -- T2
insert into test (id, value) values(3, 30); -- T1
insert into test (id, value) values(4, 42); -- T2
commit; -- T1
commit; -- T2
select * from test where value % 3 = 0; -- Either. Returns 3 => 30, 4 => 42
```

CockroachDB "serializable" prevents Anti-Dependency Cycles (G2):

```sql
begin; set transaction isolation level serializable; -- T1
begin; set transaction isolation level serializable; -- T2
select * from test where value % 3 = 0; -- T1
select * from test where value % 3 = 0; -- T2
insert into test (id, value) values(3, 30); -- T1
insert into test (id, value) values(4, 42); -- T2
commit; -- T1
commit; -- T2. Prints out "ERROR: restart transaction: TransactionRetryWithProtoRefreshError: TransactionRetryError: retry txn (RETRY_SERIALIZABLE - failed preemptive refresh)"
```

CockroachDB "serializable" prevents Anti-Dependency Cycles (G2) -- Fekete et al's example with two anti-dependency edges:

```sql
begin; set transaction isolation level serializable; -- T1
select * from test; -- T1. Shows 1 => 10, 2 => 20
begin; set transaction isolation level serializable; -- T2
update test set value = value + 5 where id = 2; -- T2
commit; -- T2
begin; set transaction isolation level serializable; -- T3
select * from test; -- T3. Shows 1 => 10, 2 => 25
commit; -- T3
update test set value = 0 where id = 1; -- T1. 
commit; -- T1. Prints out "ERROR: restart transaction: TransactionRetryWithProtoRefreshError: TransactionRetryError: retry txn (RETRY_SERIALIZABLE - failed preemptive refresh)"
```
