Testing PostgreSQL transaction isolation levels
===============================================

These tests were run with Postgres 9.3.5.

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

Postgres "read committed" prevents Write Cycles (G0) by locking updated rows:

```sql
begin; set transaction isolation level read committed; -- T1
begin; set transaction isolation level read committed; -- T2
update test set value = 11 where id = 1; -- T1
update test set value = 12 where id = 1; -- T2, BLOCKS
update test set value = 21 where id = 2; -- T1
commit; -- T1. This unblocks T2
select * from test; -- T1. Shows 1 => 11, 2 => 21
update test set value = 22 where id = 2; -- T2
commit; -- T2
select * from test; -- either. Shows 1 => 12, 2 => 22
```

Postgres "read committed" prevents Aborted Reads (G1a):

```sql
begin; set transaction isolation level read committed; -- T1
begin; set transaction isolation level read committed; -- T2
update test set value = 101 where id = 1; -- T1
select * from test; -- T2. Still shows 1 => 10
abort;  -- T1
select * from test; -- T2. Still shows 1 => 10
commit; -- T2
```

Postgres "read committed" prevents Intermediate Reads (G1b):

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

Postgres "read committed" prevents Circular Information Flow (G1c):

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

Postgres "read committed" prevents Observed Transaction Vanishes (OTV):

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

Postgres "read committed" does not prevent Predicate-Many-Preceders (PMP):

```sql
begin; set transaction isolation level read committed; -- T1
begin; set transaction isolation level read committed; -- T2
select * from test where value = 30; -- T1. Returns nothing
insert into test (id, value) values(3, 30); -- T2
commit; -- T2
select * from test where value % 3 = 0; -- T1. Returns the newly inserted row
commit; -- T1
```

Postgres "repeatable read" prevents Predicate-Many-Preceders (PMP):

```sql
begin; set transaction isolation level repeatable read; -- T1
begin; set transaction isolation level repeatable read; -- T2
select * from test where value = 30; -- T1. Returns nothing
insert into test (id, value) values(3, 30); -- T2
commit; -- T2
select * from test where value % 3 = 0; -- T1. Still returns nothing
commit; -- T1
```

Postgres "read committed" does not prevent Predicate-Many-Preceders (PMP) for write predicates -- example from Postgres documentation:

```sql
begin; set transaction isolation level read committed; -- T1
begin; set transaction isolation level read committed; -- T2
update test set value = value + 10; -- T1
delete from test where value = 20;  -- T2, BLOCKS
commit; -- T1. This unblocks T2
select * from test where value = 20; -- T2, returns 1 => 20 (despite ostensibly having been deleted)
commit; -- T2
```

Postgres "repeatable read" prevents Predicate-Many-Preceders (PMP) for write predicates -- example from Postgres documentation:

```sql
begin; set transaction isolation level repeatable read; -- T1
begin; set transaction isolation level repeatable read; -- T2
update test set value = value + 10; -- T1
delete from test where value = 20;  -- T2, BLOCKS
commit; -- T1. T2 now prints out "ERROR: could not serialize access due to concurrent update"
abort;  -- T2. There's nothing else we can do, this transaction has failed
```


Lost Update (P4)
----------------

Postgres "read committed" does not prevent Lost Update (P4):

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

Postgres "repeatable read" prevents Lost Update (P4):

```sql
begin; set transaction isolation level repeatable read; -- T1
begin; set transaction isolation level repeatable read; -- T2
select * from test where id = 1; -- T1
select * from test where id = 1; -- T2
update test set value = 11 where id = 1; -- T1
update test set value = 11 where id = 1; -- T2, BLOCKS
commit; -- T1. T2 now prints out "ERROR: could not serialize access due to concurrent update"
abort;  -- T2. There's nothing else we can do, this transaction has failed
```


Read Skew (G-single)
--------------------

Postgres "read committed" does not prevent Read Skew (G-single):

```sql
begin; set transaction isolation level read committed; -- T1
begin; set transaction isolation level read committed; -- T2
select * from test where id = 1; -- T1. Shows 1 => 10
select * from test where id = 1; -- T2
select * from test where id = 2; -- T2
update test set value = 12 where id = 1; -- T2
update test set value = 18 where id = 2; -- T2
commit; -- T2
select * from test where id = 2; -- T1. Shows 2 => 18
commit; -- T1
```

Postgres "repeatable read" prevents Read Skew (G-single):

```sql
begin; set transaction isolation level repeatable read; -- T1
begin; set transaction isolation level repeatable read; -- T2
select * from test where id = 1; -- T1. Shows 1 => 10
select * from test where id = 1; -- T2
select * from test where id = 2; -- T2
update test set value = 12 where id = 1; -- T2
update test set value = 18 where id = 2; -- T2
commit; -- T2
select * from test where id = 2; -- T1. Shows 2 => 20
commit; -- T1
```

Postgres "repeatable read" prevents Read Skew (G-single) -- test using predicate dependencies:

```sql
begin; set transaction isolation level repeatable read; -- T1
begin; set transaction isolation level repeatable read; -- T2
select * from test where value % 5 = 0; -- T1
update test set value = 12 where value = 10; -- T2
commit; -- T2
select * from test where value % 3 = 0; -- T1. Returns nothing
commit; -- T1
```

Postgres "repeatable read" prevents Read Skew (G-single) -- test using write predicate:

```sql
begin; set transaction isolation level repeatable read; -- T1
begin; set transaction isolation level repeatable read; -- T2
select * from test where id = 1; -- T1. Shows 1 => 10
select * from test; -- T2
update test set value = 12 where id = 1; -- T2
update test set value = 18 where id = 2; -- T2
commit; -- T2
delete from test where value = 20; -- T1. Prints "ERROR: could not serialize access due to concurrent update"
abort; -- T1. There's nothing else we can do, this transaction has failed
```


Write Skew (G2-item)
--------------------

Postgres "repeatable read" does not prevent Write Skew (G2-item):

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

Postgres "serializable" prevents Write Skew (G2-item):

```sql
begin; set transaction isolation level serializable; -- T1
begin; set transaction isolation level serializable; -- T2
select * from test where id in (1,2); -- T1
select * from test where id in (1,2); -- T2
update test set value = 11 where id = 1; -- T1
update test set value = 21 where id = 2; -- T2
commit; -- T1
commit; -- T2. Prints out "ERROR: could not serialize access due to read/write dependencies among transactions"
```


Anti-Dependency Cycles (G2)
---------------------------

Postgres "repeatable read" does not prevent Anti-Dependency Cycles (G2):

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

Postgres "serializable" prevents Anti-Dependency Cycles (G2):

```sql
begin; set transaction isolation level serializable; -- T1
begin; set transaction isolation level serializable; -- T2
select * from test where value % 3 = 0; -- T1
select * from test where value % 3 = 0; -- T2
insert into test (id, value) values(3, 30); -- T1
insert into test (id, value) values(4, 42); -- T2
commit; -- T1
commit; -- T2. Prints out "ERROR: could not serialize access due to read/write dependencies among transactions"
```

Postgres "serializable" prevents Anti-Dependency Cycles (G2) -- Fekete et al's example with two anti-dependency edges:

```sql
begin; set transaction isolation level serializable; -- T1
select * from test; -- T1. Shows 1 => 10, 2 => 20
begin; set transaction isolation level serializable; -- T2
update test set value = value + 5 where id = 2; -- T2
commit; -- T2
begin; set transaction isolation level serializable; -- T3
select * from test; -- T3. Shows 1 => 10, 2 => 25
commit; -- T3
update test set value = 0 where id = 1; -- T1. Prints out "ERROR: could not serialize access due to read/write dependencies among transactions"
abort; -- T1. There's nothing else we can do, this transaction has failed
```
