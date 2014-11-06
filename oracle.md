Testing Oracle transaction isolation levels
===========================================

These tests were run with Oracle 11.2.0.4.v1.

Setup (before every test case):

```sql
create table test (id number not null primary key, value number);
insert into test (id, value) values (1, 10);
insert into test (id, value) values (2, 20);
```


Read Committed basic requirements (G0, G1a, G1b, G1c)
-----------------------------------------------------

Oracle "read committed" prevents Write Cycles (G0) by locking updated rows:

```sql
set transaction isolation level read committed; -- T1
set transaction isolation level read committed; -- T2
update test set value = 11 where id = 1; -- T1
update test set value = 12 where id = 1; -- T2, BLOCKS
update test set value = 21 where id = 2; -- T1
commit; -- T1. This unblocks T2
select * from test; -- T1. Shows 1 => 11, 2 => 21
update test set value = 22 where id = 2; -- T2
commit; -- T2
select * from test; -- either. Shows 1 => 12, 2 => 22
```

Oracle "read committed" prevents Aborted Reads (G1a):

```sql
set transaction isolation level read committed; -- T1
set transaction isolation level read committed; -- T2
update test set value = 101 where id = 1; -- T1
select * from test; -- T2. Still shows 1 => 10
rollback; -- T1
select * from test; -- T2. Still shows 1 => 10
commit; -- T2
```

Oracle "read committed" prevents Intermediate Reads (G1b):

```sql
set transaction isolation level read committed; -- T1
set transaction isolation level read committed; -- T2
update test set value = 101 where id = 1; -- T1
select * from test; -- T2. Still shows 1 => 10
update test set value = 11 where id = 1; -- T1
commit; -- T1
select * from test; -- T2. Now shows 1 => 11
commit; -- T2
```

Oracle "read committed" prevents Circular Information Flow (G1c):

```sql
set transaction isolation level read committed; -- T1
set transaction isolation level read committed; -- T2
update test set value = 11 where id = 1; -- T1
update test set value = 22 where id = 2; -- T2
select * from test where id = 2; -- T1. Still shows 2 => 20
select * from test where id = 1; -- T2. Still shows 1 => 10
commit; -- T1
commit; -- T2
```


Observed Transaction Vanishes (OTV)
-----------------------------------

Oracle "read committed" prevents Observed Transaction Vanishes (OTV):

```sql
set transaction isolation level read committed; -- T1
set transaction isolation level read committed; -- T2
set transaction isolation level read committed; -- T3
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

Oracle "read committed" does not prevent Predicate-Many-Preceders (PMP):

```sql
set transaction isolation level read committed; -- T1
set transaction isolation level read committed; -- T2
select * from test where value = 30; -- T1. Returns nothing
insert into test (id, value) values(3, 30); -- T2
commit; -- T2
select * from test where mod(value, 3) = 0; -- T1. Returns the newly inserted row
commit; -- T1
```

Oracle "serializable" prevents Predicate-Many-Preceders (PMP):

```sql
set transaction isolation level serializable; -- T1
set transaction isolation level serializable; -- T2
select * from test where value = 30; -- T1. Returns nothing
insert into test (id, value) values(3, 30); -- T2
commit; -- T2
select * from test where mod(value, 3) = 0; -- T1. Still returns nothing
commit; -- T1
```

Oracle "read committed" does not prevent Predicate-Many-Preceders (PMP) for write predicates -- example from Postgres documentation:

```sql
set transaction isolation level read committed; -- T1
set transaction isolation level read committed; -- T2
update test set value = value + 10; -- T1
select * from test; -- T2. Returns 1 => 10, 2 => 20
delete from test where value = 20;  -- T2, BLOCKS
commit; -- T1. This unblocks T2
select * from test; -- T2. Returns 2 => 30
commit; -- T2
```

Oracle "serializable" prevents Predicate-Many-Preceders (PMP) for write predicates -- example from Postgres documentation:

```sql
set transaction isolation level serializable; -- T1
set transaction isolation level serializable; -- T2
update test set value = value + 10; -- T1
delete from test where value = 20;  -- T2, BLOCKS
commit;   -- T1. T2 now prints "ORA-08177: can't serialize access for this transaction"
rollback; -- T2
```


Lost Update (P4)
----------------

Oracle "read committed" does not prevent Lost Update (P4):

```sql
set transaction isolation level read committed; -- T1
set transaction isolation level read committed; -- T2
select * from test where id = 1; -- T1
select * from test where id = 1; -- T2
update test set value = 11 where id = 1; -- T1
update test set value = 11 where id = 1; -- T2, BLOCKS
commit; -- T1. This unblocks T2, so T1's update is overwritten
commit; -- T2
```

Oracle "serializable" prevents Lost Update (P4):

```sql
set transaction isolation level serializable; -- T1
set transaction isolation level serializable; -- T2
select * from test where id = 1; -- T1
select * from test where id = 1; -- T2
update test set value = 11 where id = 1; -- T1
update test set value = 11 where id = 1; -- T2, BLOCKS
commit;   -- T1. T2 now prints out "ORA-08177: can't serialize access for this transaction"
rollback; -- T2
```


Read Skew (G-single)
--------------------

Oracle "read committed" does not prevent Read Skew (G-single):

```sql
set transaction isolation level read committed; -- T1
set transaction isolation level read committed; -- T2
select * from test where id = 1; -- T1. Shows 1 => 10
select * from test where id = 1; -- T2
select * from test where id = 2; -- T2
update test set value = 12 where id = 1; -- T2
update test set value = 18 where id = 2; -- T2
commit; -- T2
select * from test where id = 2; -- T1. Shows 2 => 18
commit; -- T1
```

Oracle "serializable" prevents Read Skew (G-single):

```sql
set transaction isolation level serializable; -- T1
set transaction isolation level serializable; -- T2
select * from test where id = 1; -- T1. Shows 1 => 10
select * from test where id = 1; -- T2
select * from test where id = 2; -- T2
update test set value = 12 where id = 1; -- T2
update test set value = 18 where id = 2; -- T2
commit; -- T2
select * from test where id = 2; -- T1. Shows 2 => 20
commit; -- T1
```

Oracle "serializable" prevents Read Skew (G-single) -- test using predicate dependencies:

```sql
set transaction isolation level serializable; -- T1
set transaction isolation level serializable; -- T2
select * from test where mod(value, 5) = 0; -- T1
update test set value = 12 where value = 10; -- T2
commit; -- T2
select * from test where mod(value, 3) = 0; -- T1. Returns nothing
commit; -- T1
```

Oracle "serializable" prevents Read Skew (G-single) -- test using write predicate:

```sql
set transaction isolation level serializable; -- T1
set transaction isolation level serializable; -- T2
select * from test where id = 1; -- T1. Shows 1 => 10
select * from test; -- T2
update test set value = 12 where id = 1; -- T2
update test set value = 18 where id = 2; -- T2
commit; -- T2
delete from test where value = 20; -- T1. Prints "ORA-08177: can't serialize access for this transaction"
rollback; -- T1
```


Write Skew (G2-item)
--------------------

Oracle "serializable" does not prevent Write Skew (G2-item):

```sql
set transaction isolation level serializable; -- T1
set transaction isolation level serializable; -- T2
select * from test where id in (1,2); -- T1
select * from test where id in (1,2); -- T2
update test set value = 11 where id = 1; -- T1
update test set value = 21 where id = 2; -- T2
commit; -- T1
commit; -- T2
select * from test; -- T1. Returns 1 => 11, 2 => 21
```


Anti-Dependency Cycles (G2)
---------------------------

Oracle "read committed" does not prevent Anti-Dependency Cycles (G2):

```sql
set transaction isolation level read committed; -- T1
set transaction isolation level read committed; -- T2
select * from test where mod(value, 3) = 0; -- T1
select * from test where mod(value, 3) = 0; -- T2
insert into test (id, value) values(3, 30); -- T1
insert into test (id, value) values(4, 42); -- T2
commit; -- T1
commit; -- T2
select * from test where mod(value, 3) = 0; -- T1. Returns 3 => 30, 4 => 42
```

Oracle "serializable" prevents Anti-Dependency Cycles (G2) in the special case where the dependency predicates are the same
(note: this happened once, but now I can't reproduce it, so I'm not sure what's going on here):

```sql
set transaction isolation level serializable; -- T1
set transaction isolation level serializable; -- T2
select * from test where mod(value, 3) = 0; -- T1
select * from test where mod(value, 3) = 0; -- T2
insert into test (id, value) values(3, 30); -- T1
insert into test (id, value) values(4, 42); -- T2. Prints "ORA-08177: can't serialize access for this transaction"
commit; -- T1
rollback; -- T2
select * from test where mod(value, 3) = 0; -- T1. Returns 3 => 30
```

Oracle "serializable" does not prevent Anti-Dependency Cycles (G2) in general:

```sql
set transaction isolation level serializable; -- T1
set transaction isolation level serializable; -- T2
select * from test where mod(value, 3) = 0; -- T1
select * from test where mod(value, 5) = 0; -- T2
insert into test (id, value) values(3, 30); -- T1
insert into test (id, value) values(4, 60); -- T2
commit; -- T1
commit; -- T2
select * from test where mod(value, 3) = 0; -- T1. Returns 3 => 30, 4 => 60
```

However, Oracle "serializable" does prevent this case of Anti-Dependency Cycles (G2) -- Fekete et al's example with two anti-dependency edges:

```sql
set transaction isolation level serializable; -- T1
select * from test; -- T1. Shows 1 => 10, 2 => 20
set transaction isolation level serializable; -- T2
update test set value = value + 5 where id = 2; -- T2
commit; -- T2
set transaction isolation level serializable; -- T3
select * from test; -- T3. Shows 1 => 10, 2 => 25
commit; -- T3
update test set value = 0 where id = 1; -- T1. Prints out "ORA-08177: can't serialize access for this transaction"
rollback; -- T1
```


Practical notes
---------------

If you want to run these tests and don't happen to have an Oracle license lying around, you can
bring up an Oracle instance on Amazon RDS:

1. On the [RDS web console](https://us-west-2.console.aws.amazon.com/rds/home?region=us-west-2),
   launch an oracle-se1 instance. ([Instructions](http://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/CHAP_GettingStarted.CreatingConnecting.Oracle.html))
   * Say no to provisioned IOPS and multi-AZ. 
   * License model: license-included
   * Instance class: db.m1.small
   * Instance identifier: oracletest
   * Master username: oracletest
   * Password: oracletest
   * Database name: ORCL
   * Database port: 1521
   * You may need to allow access to incoming port 1521 in the security group settings.
2. When RDS has launched the instance (which takes a while), you should get an endpoint. Note the hostname, which
   looks something like `oracletest.a1b2c4d5e5f6.us-west-2.rds.amazonaws.com`.
3. Download [Instant Client](http://www.oracle.com/technetwork/topics/linuxx86-64soft-092277.html) for Linux.
   ([Instructions](http://docs.oracle.com/cd/B19306_01/server.102/b14357/ape.htm))
   You need to get the same version of the client as you launched (e.g. 11.2.0.4).
   You need two packages: "Instant Client Package - Basic" and "Instant Client Package - SQL*Plus".
   Get the zip version (not the rpm version). The files should be called something like
   `instantclient-basic-linux.x64-11.2.0.4.0.zip` and `instantclient-sqlplus-linux.x64-11.2.0.4.0.zip`.
4. In the same directory as the instantclient zip files, create a file `Dockerfile` with contents:
   ```
   FROM ubuntu:14.04
   RUN apt-get update && apt-get install -y unzip libaio1
   RUN mkdir /oracle
   COPY instantclient-*.zip /oracle/
   COPY oracletest /oracle/
   RUN cd /oracle && unzip instantclient-basic-*.zip && unzip instantclient-sqlplus-*.zip
   ```
5. In that same directory, create a file `oracletest` with chmod 755 and contents:
   ```bash
   #!/bin/bash
   cd /oracle/instantclient_11_2
   LD_LIBRARY_PATH=/oracle/instantclient_11_2:${LD_LIBRARY_PATH} \
      ./sqlplus "oracletest@(DESCRIPTION=(ADDRESS=(PROTOCOL=TCP)(HOST=$1)(PORT=1521))(CONNECT_DATA=(SID=ORCL)))"
   ```
6. Install [Docker](https://www.docker.com/) if you don't already have it, cd to that directory and do
   `docker build -t=oracletest .`
7. To connect to the database, run this (substituting your own instance hostname)
   `docker run -t -i oracletest /oracle/oracletest oracletest.a1b2c4d5e5f6.us-west-2.rds.amazonaws.com`
   and when prompted, enter the password `oracletest`.
