Testing MS SQL Server transaction isolation levels
==================================================

These tests were run with SQL Server 11.00.2100.60.v1.

Setup:

```sql
create database test_lock;
create database test_snap1;
create database test_snap2;
alter database test_lock  set read_committed_snapshot  off;
alter database test_lock  set allow_snapshot_isolation off;
alter database test_snap1 set read_committed_snapshot  on;
alter database test_snap1 set allow_snapshot_isolation off;
alter database test_snap2 set read_committed_snapshot  off;
alter database test_snap2 set allow_snapshot_isolation on;
create table test_lock.dbo.test  (id int primary key, value int);
create table test_snap1.dbo.test (id int primary key, value int);
create table test_snap2.dbo.test (id int primary key, value int);
insert into test_lock.dbo.test  (id, value) values(1, 10), (2, 20);
insert into test_snap1.dbo.test (id, value) values(1, 10), (2, 20);
insert into test_snap2.dbo.test (id, value) values(1, 10), (2, 20);
```


Read Committed basic requirements (G0, G1a, G1b, G1c)
-----------------------------------------------------

SQL Server "read uncommitted" prevents Write Cycles (G0) by locking updated rows:

```sql
set transaction isolation level read uncommitted; begin transaction; -- T1
set transaction isolation level read uncommitted; begin transaction; -- T2
update test_lock.dbo.test set value = 11 where id = 1; -- T1
update test_lock.dbo.test set value = 12 where id = 1; -- T2, BLOCKS
update test_lock.dbo.test set value = 21 where id = 2; -- T1
commit; -- T1. This unblocks T2
select * from test_lock.dbo.test; -- T1. Shows 1 => 12, 2 => 21
update test_lock.dbo.test set value = 22 where id = 2; -- T2
commit; -- T2
select * from test_lock.dbo.test; -- either. Shows 1 => 12, 2 => 22
```

SQL Server "read uncommitted" does not prevent Aborted Reads (G1a):

```sql
set transaction isolation level read uncommitted; begin transaction; -- T1
set transaction isolation level read uncommitted; begin transaction; -- T2
update test_lock.dbo.test set value = 101 where id = 1; -- T1
select * from test_lock.dbo.test; -- T2. Shows 1 => 101
rollback; -- T1
select * from test_lock.dbo.test; -- T2. Shows 1 => 10 again
commit; -- T2
```

SQL Server locking "read committed" prevents Aborted Reads (G1a):

```sql
set transaction isolation level read committed; begin transaction; -- T1
set transaction isolation level read committed; begin transaction; -- T2
update test_lock.dbo.test set value = 101 where id = 1; -- T1
select * from test_lock.dbo.test; -- T2, BLOCKS
rollback; -- T1. Unblocks T2, which now shows 1 => 10, 2 => 20
commit; -- T2
```

SQL Server snapshot "read committed" prevents Aborted Reads (G1a):

```sql
set transaction isolation level read committed; begin transaction; -- T1
set transaction isolation level read committed; begin transaction; -- T2
update test_snap1.dbo.test set value = 101 where id = 1; -- T1
select * from test_snap1.dbo.test; -- T2. Shows 1 => 10
rollback; -- T1. T2 now shows 1 => 10, 2 => 20
select * from test_snap1.dbo.test; -- T2. Still shows 1 => 10
commit; -- T2
```

SQL Server "read uncommitted" does not prevent Intermediate Reads (G1b):

```sql
set transaction isolation level read uncommitted; begin transaction; -- T1
set transaction isolation level read uncommitted; begin transaction; -- T2
update test_lock.dbo.test set value = 101 where id = 1; -- T1
select * from test_lock.dbo.test; -- T2. Shows 1 => 101
update test_lock.dbo.test set value = 11 where id = 1; -- T1
commit; -- T1
select * from test_lock.dbo.test; -- T2. Now shows 1 => 11
commit; -- T2
```

SQL Server locking "read committed" prevents Intermediate Reads (G1b):

```sql
set transaction isolation level read committed; begin transaction; -- T1
set transaction isolation level read committed; begin transaction; -- T2
update test_lock.dbo.test set value = 101 where id = 1; -- T1
select * from test_lock.dbo.test; -- T2, BLOCKS
update test_lock.dbo.test set value = 11 where id = 1; -- T1
commit; -- T1. Unblocks T2, which shows 1 => 11
commit; -- T2
```

SQL Server snapshot "read committed" prevents Intermediate Reads (G1b):

```sql
set transaction isolation level read committed; begin transaction; -- T1
set transaction isolation level read committed; begin transaction; -- T2
update test_snap1.dbo.test set value = 101 where id = 1; -- T1
select * from test_snap1.dbo.test; -- T2. Still shows 1 => 10
update test_snap1.dbo.test set value = 11 where id = 1; -- T1
commit; -- T1
select * from test_snap1.dbo.test; -- T2. Now shows 1 => 11
commit; -- T2
```

SQL Server "read uncommitted" does not prevent Circular Information Flow (G1c):

```sql
set transaction isolation level read uncommitted; begin transaction; -- T1
set transaction isolation level read uncommitted; begin transaction; -- T2
update test_lock.dbo.test set value = 11 where id = 1; -- T1
update test_lock.dbo.test set value = 22 where id = 2; -- T2
select * from test_lock.dbo.test where id = 2; -- T1. Shows 2 => 22
select * from test_lock.dbo.test where id = 1; -- T2. Shows 1 => 11
commit; -- T1
commit; -- T2
```

SQL Server locking "read committed" prevents Circular Information Flow (G1c):

```sql
set transaction isolation level read committed; begin transaction; -- T1
set transaction isolation level read committed; begin transaction; -- T2
update test_lock.dbo.test set value = 11 where id = 1; -- T1
update test_lock.dbo.test set value = 22 where id = 2; -- T2
select * from test_lock.dbo.test where id = 2; -- T1, BLOCKS
select * from test_lock.dbo.test where id = 1; -- T2, prints "Transaction was deadlocked on lock resources with another process and has been chosen as the deadlock victim. Rerun the transaction."
commit; -- T1
```

SQL Server snapshot "read committed" prevents Circular Information Flow (G1c):

```sql
set transaction isolation level read committed; begin transaction; -- T1
set transaction isolation level read committed; begin transaction; -- T2
update test_snap1.dbo.test set value = 11 where id = 1; -- T1
update test_snap1.dbo.test set value = 22 where id = 2; -- T2
select * from test_snap1.dbo.test where id = 2; -- T1. Still shows 2 => 20
select * from test_snap1.dbo.test where id = 1; -- T2. Still shows 1 => 10
commit; -- T1
commit; -- T2
```


Observed Transaction Vanishes (OTV)
-----------------------------------

SQL Server "read uncommitted" does not prevent Observed Transaction Vanishes (OTV):

```sql
set transaction isolation level read uncommitted; begin transaction; -- T1
set transaction isolation level read uncommitted; begin transaction; -- T2
set transaction isolation level read uncommitted; begin transaction; -- T3
update test_lock.dbo.test set value = 11 where id = 1; -- T1
update test_lock.dbo.test set value = 19 where id = 2; -- T1
update test_lock.dbo.test set value = 12 where id = 1; -- T2. BLOCKS
commit; -- T1. This unblocks T2
select * from test_lock.dbo.test; -- T3. Shows 1 => 12, 2 => 19
update test_lock.dbo.test set value = 18 where id = 2; -- T2
select * from test_lock.dbo.test; -- T3. Shows 1 => 12, 2 => 18
commit; -- T2
commit; -- T3
```

SQL Server locking "read committed" prevents Observed Transaction Vanishes (OTV):

```sql
set transaction isolation level read committed; begin transaction; -- T1
set transaction isolation level read committed; begin transaction; -- T2
set transaction isolation level read committed; begin transaction; -- T3
update test_lock.dbo.test set value = 11 where id = 1; -- T1
update test_lock.dbo.test set value = 19 where id = 2; -- T1
update test_lock.dbo.test set value = 12 where id = 1; -- T2. BLOCKS
commit; -- T1. This unblocks T2
select * from test_lock.dbo.test; -- T3. BLOCKS
update test_lock.dbo.test set value = 18 where id = 2; -- T2
commit; -- T2. Unblocks T3, which shows 1 => 12, 2 => 18
commit; -- T3
```

SQL Server snapshot "read committed" prevents Observed Transaction Vanishes (OTV):

```sql
set transaction isolation level read committed; begin transaction; -- T1
set transaction isolation level read committed; begin transaction; -- T2
set transaction isolation level read committed; begin transaction; -- T3
update test_snap1.dbo.test set value = 11 where id = 1; -- T1
update test_snap1.dbo.test set value = 19 where id = 2; -- T1
update test_snap1.dbo.test set value = 12 where id = 1; -- T2. BLOCKS
commit; -- T1. This unblocks T2
select * from test_snap1.dbo.test; -- T3. Shows 1 => 11, 2 => 19
update test_snap1.dbo.test set value = 18 where id = 2; -- T2
select * from test_snap1.dbo.test; -- T3. Shows 1 => 11, 2 => 19
commit; -- T2
select * from test_snap1.dbo.test; -- T3. Shows 1 => 12, 2 => 18
commit; -- T3
```


Predicate-Many-Preceders (PMP)
------------------------------

SQL Server locking "read committed" does not prevent Predicate-Many-Preceders (PMP):

```sql
set transaction isolation level read committed; begin transaction; -- T1
set transaction isolation level read committed; begin transaction; -- T2
select * from test_lock.dbo.test where value = 30; -- T1. Returns nothing
insert into test_lock.dbo.test (id, value) values(3, 30); -- T2
commit; -- T2
select * from test_lock.dbo.test where value % 3 = 0; -- T1. Returns the newly inserted row
commit; -- T1
```

SQL Server snapshot "read committed" does not prevent Predicate-Many-Preceders (PMP):

```sql
set transaction isolation level read committed; begin transaction; -- T1
set transaction isolation level read committed; begin transaction; -- T2
select * from test_snap1.dbo.test where value = 30; -- T1. Returns nothing
insert into test_snap1.dbo.test (id, value) values(3, 30); -- T2
commit; -- T2
select * from test_snap1.dbo.test where value % 3 = 0; -- T1. Returns the newly inserted row
commit; -- T1
```

SQL Server "repeatable read" does not prevent Predicate-Many-Preceders (PMP) for read predicates:

```sql
set transaction isolation level repeatable read; begin transaction; -- T1
set transaction isolation level repeatable read; begin transaction; -- T2
select * from test_lock.dbo.test where value = 30; -- T1. Returns nothing
insert into test_lock.dbo.test (id, value) values(3, 30); -- T2
commit; -- T2
select * from test_snap1.dbo.test where value % 3 = 0; -- T1. Returns the newly inserted row
commit; -- T1
```

SQL Server "snapshot" prevents Predicate-Many-Preceders (PMP) for read predicates:

```sql
set transaction isolation level snapshot; begin transaction; -- T1
set transaction isolation level snapshot; begin transaction; -- T2
select * from test_snap2.dbo.test where value = 30; -- T1. Returns nothing
insert into test_snap2.dbo.test (id, value) values(3, 30); -- T2
commit; -- T2
select * from test_snap2.dbo.test where value % 3 = 0; -- T1. Still returns nothing
commit; -- T1
```

SQL Server "serializable" prevents Predicate-Many-Preceders (PMP) for read predicates:

```sql
set transaction isolation level serializable; begin transaction; -- T1
set transaction isolation level serializable; begin transaction; -- T2
select * from test_lock.dbo.test where value = 30; -- T1. Returns nothing
insert into test_lock.dbo.test (id, value) values(3, 30); -- T2, BLOCKS
select * from test_lock.dbo.test where value % 3 = 0; -- T1. Still returns nothing
commit; -- T1. Unblocks T2
commit; -- T2
```

SQL Server locking "read committed" does not prevent Predicate-Many-Preceders (PMP) for existing items -- example from Postgres documentation:

```sql
set transaction isolation level read committed; begin transaction; -- T1
set transaction isolation level read committed; begin transaction; -- T2
select * from test_lock.dbo.test; -- T2, returns 1 => 10, 2 => 20
update test_lock.dbo.test set value = value + 10; -- T1
select * from test_lock.dbo.test; -- T2, BLOCKS
commit; -- T1. This unblocks T2 which now returns 1 => 20, 2 => 30
delete from test_lock.dbo.test where value = 20; -- T2
select * from test_lock.dbo.test; -- T2. Returns 2 => 30
commit; -- T2
```

SQL Server snapshot "read committed" does not prevent Predicate-Many-Preceders (PMP) for existing items -- example from Postgres documentation:

```sql
set transaction isolation level read committed; begin transaction; -- T1
set transaction isolation level read committed; begin transaction; -- T2
update test_snap1.dbo.test set value = value + 10; -- T1
select * from test_snap1.dbo.test where value = 20; -- T2. Returns 2 => 20
delete from test_snap1.dbo.test where value = 20; -- T2, BLOCKS
commit; -- T1. This unblocks T2
select * from test_snap1.dbo.test; -- T2. Returns 2 => 30
commit; -- T2
```

SQL Server "repeatable read" prevents Predicate-Many-Preceders (PMP) for existing items -- example from Postgres documentation:

```sql
set transaction isolation level repeatable read; begin transaction; -- T1
set transaction isolation level repeatable read; begin transaction; -- T2
select * from test_lock.dbo.test; -- T2, returns 1 => 10, 2 => 20
update test_lock.dbo.test set value = value + 10; -- T1. BLOCKS
delete from test_lock.dbo.test where value = 20; -- T2. Prints "Transaction (Process ID 87) was deadlocked on lock resources with another process and has been chosen as the deadlock victim. Rerun the transaction."
commit; -- T1
```

SQL Server "snapshot" prevents Predicate-Many-Preceders (PMP) for write predicates -- example from Postgres documentation:

```sql
set transaction isolation level snapshot; begin transaction; -- T1
set transaction isolation level snapshot; begin transaction; -- T2
update test_snap2.dbo.test set value = value + 10; -- T1
select * from test_snap2.dbo.test where value = 20; -- T2. Returns 2 => 20
delete from test_snap2.dbo.test where value = 20; -- T2, BLOCKS
commit; -- T1. T2 now prints "Snapshot isolation transaction aborted due to update conflict. You cannot use snapshot isolation to access table 'dbo.test' directly or indirectly in database 'test_snap2' to update, delete, or insert the row that has been modified or deleted by another transaction. Retry the transaction or change the isolation level for the update/delete statement."
```

SQL Server "serializable" prevents Predicate-Many-Preceders (PMP) for write predicates -- example from Postgres documentation:

```sql
set transaction isolation level serializable; begin transaction; -- T1
set transaction isolation level serializable; begin transaction; -- T2
select * from test_lock.dbo.test where value = 20; -- T2, returns 2 => 20
update test_lock.dbo.test set value = value + 10; -- T1, BLOCKS
delete from test_lock.dbo.test where value = 20; -- T2, prints "Transaction was deadlocked on lock resources with another process and has been chosen as the deadlock victim. Rerun the transaction."
commit; -- T1
```


Lost Update (P4)
----------------

SQL Server locking "read committed" does not prevent Lost Update (P4):

```sql
set transaction isolation level read committed; begin transaction; -- T1
set transaction isolation level read committed; begin transaction; -- T2
select * from test_lock.dbo.test where id = 1; -- T1
select * from test_lock.dbo.test where id = 1; -- T2
update test_lock.dbo.test set value = 11 where id = 1; -- T1
update test_lock.dbo.test set value = 11 where id = 1; -- T2, BLOCKS
commit; -- T1. Unblocks T2
commit; -- T2
```

SQL Server snapshot "read committed" does not prevent Lost Update (P4):

```sql
set transaction isolation level read committed; begin transaction; -- T1
set transaction isolation level read committed; begin transaction; -- T2
select * from test_snap1.dbo.test where id = 1; -- T1
select * from test_snap1.dbo.test where id = 1; -- T2
update test_snap1.dbo.test set value = 11 where id = 1; -- T1
update test_snap1.dbo.test set value = 11 where id = 1; -- T2, BLOCKS
commit; -- T1. Unblocks T2
commit; -- T2
```

SQL Server "repeatable read" prevents Lost Update (P4):

```sql
set transaction isolation level repeatable read; begin transaction; -- T1
set transaction isolation level repeatable read; begin transaction; -- T2
select * from test_lock.dbo.test where id = 1; -- T1
select * from test_lock.dbo.test where id = 1; -- T2
update test_lock.dbo.test set value = 11 where id = 1; -- T1, BLOCKS
update test_lock.dbo.test set value = 11 where id = 1; -- T2, prints "Transaction was deadlocked on lock resources with another process and has been chosen as the deadlock victim. Rerun the transaction."
commit; -- T1
```

SQL Server "snapshot" prevents Lost Update (P4):

```sql
set transaction isolation level snapshot; begin transaction; -- T1
set transaction isolation level snapshot; begin transaction; -- T2
select * from test_snap2.dbo.test where id = 1; -- T1
select * from test_snap2.dbo.test where id = 1; -- T2
update test_snap2.dbo.test set value = 11 where id = 1; -- T1
update test_snap2.dbo.test set value = 11 where id = 1; -- T2, BLOCKS
commit; -- T1. Causes T2 to print out "Snapshot isolation transaction aborted due to update conflict. You cannot use snapshot isolation to access table 'dbo.test' directly or indirectly in database 'test_snap2' to update, delete, or insert the row that has been modified or deleted by another transaction. Retry the transaction or change the isolation level for the update/delete statement."
```


Read Skew (G-single)
--------------------

SQL Server locking "read committed" does not prevent Read Skew (G-single):

```sql
set transaction isolation level read committed; begin transaction; -- T1
set transaction isolation level read committed; begin transaction; -- T2
select * from test_lock.dbo.test where id = 1; -- T1. Shows 1 => 10
select * from test_lock.dbo.test where id = 1; -- T2
select * from test_lock.dbo.test where id = 2; -- T2
update test_lock.dbo.test set value = 12 where id = 1; -- T2
update test_lock.dbo.test set value = 18 where id = 2; -- T2
commit; -- T2
select * from test_lock.dbo.test where id = 2; -- T1. Shows 2 => 18
commit; -- T1
```

SQL Server snapshot "read committed" does not prevent Read Skew (G-single):

```sql
set transaction isolation level read committed; begin transaction; -- T1
set transaction isolation level read committed; begin transaction; -- T2
select * from test_snap1.dbo.test where id = 1; -- T1. Shows 1 => 10
select * from test_snap1.dbo.test where id = 1; -- T2
select * from test_snap1.dbo.test where id = 2; -- T2
update test_snap1.dbo.test set value = 12 where id = 1; -- T2
update test_snap1.dbo.test set value = 18 where id = 2; -- T2
commit; -- T2
select * from test_snap1.dbo.test where id = 2; -- T1. Shows 2 => 18
commit; -- T1
```

SQL Server "repeatable read" prevents Read Skew (G-single) on a read-only transaction:

```sql
set transaction isolation level repeatable read; begin transaction; -- T1
set transaction isolation level repeatable read; begin transaction; -- T2
select * from test_lock.dbo.test where id = 1; -- T1. Shows 1 => 10
select * from test_lock.dbo.test where id = 1; -- T2
select * from test_lock.dbo.test where id = 2; -- T2
update test_lock.dbo.test set value = 12 where id = 1; -- T2, BLOCKS
select * from test_lock.dbo.test where id = 2; -- T1. Shows 2 => 20
commit; -- T1. Unblocks T2
update test_lock.dbo.test set value = 18 where id = 2; -- T2
commit; -- T2
```

SQL Server "snapshot" prevents Read Skew (G-single) on a read-only transaction:

```sql
set transaction isolation level snapshot; begin transaction; -- T1
set transaction isolation level snapshot; begin transaction; -- T2
select * from test_snap2.dbo.test where id = 1; -- T1. Shows 1 => 10
select * from test_snap2.dbo.test where id = 1; -- T2
select * from test_snap2.dbo.test where id = 2; -- T2
update test_snap2.dbo.test set value = 12 where id = 1; -- T2
update test_snap2.dbo.test set value = 18 where id = 2; -- T2
commit; -- T2
select * from test_snap2.dbo.test where id = 2; -- T1. Shows 2 => 20
commit; -- T1
```

SQL Server "repeatable read" does not prevent Read Skew (G-single) on predicate dependencies:

```sql
set transaction isolation level repeatable read; begin transaction; -- T1
set transaction isolation level repeatable read; begin transaction; -- T2
select * from test_lock.dbo.test where value % 5 = 0; -- T1
insert into test_lock.dbo.test (id, value) values (3, 30); -- T2
commit; -- T2
select * from test_lock.dbo.test where value % 3 = 0; -- T1. Returns 3 => 30
commit; -- T1
```

SQL Server "snapshot" prevents Read Skew (G-single) on predicate dependencies:

```sql
set transaction isolation level snapshot; begin transaction; -- T1
set transaction isolation level snapshot; begin transaction; -- T2
select * from test_snap2.dbo.test where value % 5 = 0; -- T1
insert into test_snap2.dbo.test (id, value) values (3, 30); -- T2
commit; -- T2
select * from test_snap2.dbo.test where value % 3 = 0; -- T1. Returns nothing
commit; -- T1
```

SQL Server "serializable" prevents Read Skew (G-single) on predicate dependencies:

```sql
set transaction isolation level serializable; begin transaction; -- T1
set transaction isolation level serializable; begin transaction; -- T2
select * from test_lock.dbo.test where value % 5 = 0; -- T1
insert into test_lock.dbo.test (id, value) values (3, 30); -- T2, BLOCKS
select * from test_lock.dbo.test where value % 3 = 0; -- T1. Returns nothing
commit; -- T1. Unblocks T2
commit; -- T2
```

SQL Server "repeatable read" prevents Read Skew (G-single) on a write predicate:

```sql
set transaction isolation level repeatable read; begin transaction; -- T1
set transaction isolation level repeatable read; begin transaction; -- T2
select * from test_lock.dbo.test where id = 1; -- T1. Shows 1 => 10
select * from test_lock.dbo.test; -- T2
update test_lock.dbo.test set value = 12 where id = 1; -- T2, BLOCKS
delete from test_lock.dbo.test where value = 20; -- T1. Prints "Transaction was deadlocked on lock resources with another process and has been chosen as the deadlock victim. Rerun the transaction."
update test_lock.dbo.test set value = 18 where id = 2; -- T2
commit; -- T2
```

SQL Server "snapshot" prevents Read Skew (G-single) on a write predicate:

```sql
set transaction isolation level snapshot; begin transaction; -- T1
set transaction isolation level snapshot; begin transaction; -- T2
select * from test_snap2.dbo.test where id = 1; -- T1. Shows 1 => 10
select * from test_snap2.dbo.test; -- T2
update test_snap2.dbo.test set value = 12 where id = 1; -- T2
update test_snap2.dbo.test set value = 18 where id = 2; -- T2
commit; -- T2
delete from test_snap2.dbo.test where value = 20; -- T1. Prints "Snapshot isolation transaction aborted due to update conflict. You cannot use snapshot isolation to access table 'dbo.test' directly or indirectly in database 'test_snap2' to update, delete, or insert the row that has been modified or deleted by another transaction. Retry the transaction or change the isolation level for the update/delete statement."
```


Write Skew (G2-item)
--------------------

SQL Server "repeatable read" prevents Write Skew (G2-item):

```sql
set transaction isolation level repeatable read; begin transaction; -- T1
set transaction isolation level repeatable read; begin transaction; -- T2
select * from test_lock.dbo.test where id in (1,2); -- T1
select * from test_lock.dbo.test where id in (1,2); -- T2
update test_lock.dbo.test set value = 11 where id = 1; -- T1. BLOCKS
update test_lock.dbo.test set value = 21 where id = 2; -- T2. Prints "Transaction was deadlocked on lock resources with another process and has been chosen as the deadlock victim. Rerun the transaction."
commit; -- T1
```

SQL Server "snapshot" does not prevent Write Skew (G2-item):

```sql
set transaction isolation level snapshot; begin transaction; -- T1
set transaction isolation level snapshot; begin transaction; -- T2
select * from test_snap2.dbo.test where id in (1,2); -- T1
select * from test_snap2.dbo.test where id in (1,2); -- T2
update test_snap2.dbo.test set value = 11 where id = 1; -- T1
update test_snap2.dbo.test set value = 21 where id = 2; -- T2
commit; -- T1
commit; -- T2
```


Anti-Dependency Cycles (G2)
---------------------------

SQL Server "repeatable read" does not prevent Anti-Dependency Cycles (G2):

```sql
set transaction isolation level repeatable read; begin transaction; -- T1
set transaction isolation level repeatable read; begin transaction; -- T2
select * from test_lock.dbo.test where value % 3 = 0; -- T1
select * from test_lock.dbo.test where value % 3 = 0; -- T2
insert into test_lock.dbo.test (id, value) values(3, 30); -- T1
insert into test_lock.dbo.test (id, value) values(4, 42); -- T2
commit; -- T1
commit; -- T2
select * from test_lock.dbo.test where value % 3 = 0; -- Either. Returns 3 => 30, 4 => 42
```

SQL Server "snapshot" does not prevent Anti-Dependency Cycles (G2):

```sql
set transaction isolation level snapshot; begin transaction; -- T1
set transaction isolation level snapshot; begin transaction; -- T2
select * from test_snap2.dbo.test where value % 3 = 0; -- T1
select * from test_snap2.dbo.test where value % 3 = 0; -- T2
insert into test_snap2.dbo.test (id, value) values(3, 30); -- T1
insert into test_snap2.dbo.test (id, value) values(4, 42); -- T2
commit; -- T1
commit; -- T2
select * from test_snap2.dbo.test where value % 3 = 0; -- Either. Returns 3 => 30, 4 => 42
```

SQL Server "serializable" prevents Anti-Dependency Cycles (G2):

```sql
set transaction isolation level serializable; begin transaction; -- T1
set transaction isolation level serializable; begin transaction; -- T2
select * from test_lock.dbo.test where value % 3 = 0; -- T1
select * from test_lock.dbo.test where value % 3 = 0; -- T2
insert into test_lock.dbo.test (id, value) values(3, 30); -- T1. BLOCKS
insert into test_lock.dbo.test (id, value) values(4, 42); -- T2. Prints "Transaction was deadlocked on lock resources with another process and has been chosen as the deadlock victim. Rerun the transaction."
commit; -- T1
```

SQL Server "serializable" prevents Anti-Dependency Cycles (G2) -- Fekete et al's example with two anti-dependency edges:

```sql
set transaction isolation level serializable; begin transaction; -- T1
select * from test_lock.dbo.test; -- T1. Shows 1 => 10, 2 => 20
set transaction isolation level serializable; begin transaction; -- T2
update test_lock.dbo.test set value = value + 5 where id = 2; -- T2, BLOCKS
set transaction isolation level serializable; begin transaction; -- T3
select * from test_lock.dbo.test; -- T3, BLOCKS (eventually shows 1 => 10, 2 => 20)
update test_lock.dbo.test set value = 0 where id = 1; -- T1, aborts with deadlock error, unblocks T2
commit; -- T2, unblocks T3
commit; -- T3
```


Testing SQL Server on AWS
-------------------------

If you want to run these tests and don't happen to have a SQL Server license lying around, you can
bring up a SQL Server instance on Amazon RDS.

On the [RDS web console](https://us-west-2.console.aws.amazon.com/rds/home?region=us-west-2),
launch a sqlserver-se instance with the following settings:

* Say no to provisioned IOPS and multi-AZ. 
* License model: license-included
* Instance class: db.m1.small
* Instance identifier: sqlservertest
* Master username: sqlservertest
* Password: sqlservertest
* Database port: 1433
* You may need to allow access to incoming port 1433 in the security group settings.

On another Windows machine (or an EC2 Windows VM), download and install
[SQL Server Management Studio](http://www.microsoft.com/en-gb/download/details.aspx?id=29062).
You can then connect to SQL Server using the Management Studio UI
([Instructions](http://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/CHAP_GettingStarted.CreatingConnecting.SQLServer.html)),
or on the command line using `sqlcmd`:

```
cd 'C:\Program Files\Microsoft SQL Server\110\Tools\Binn'
.\sqlcmd -U sqlservertest -P sqlservertest -S sqlservertest.a1b2c3d4e5f6.us-west-2.rds.amazonaws.com -d testdb
```

NOTE: with `sqlcmd` you have to type `GO` (on a new line) after every SQL command so that it gets sent to the server.
