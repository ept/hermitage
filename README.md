Transaction isolation level test suite
======================================

TODO description.

| DBMS          | So-called isolation level    | Actual isolation   | G0 | G1a | G1b | G1c | OTV | PMP | P4 | G-single | G2-item | G2   |
|:--------------|:---------------------------  |:-------------------|:--:|:---:|:---:|:---:|:---:|:---:|:--:|:--------:|:-------:|:----:|
| PostgreSQL    | "read committed" ★           | MAV                | ✓  | ✓   | ✓   | ✓   | ✓   | —   | —  | —        | —       | —    |
|               | "repeatable read"            | snapshot isolation | ✓  | ✓   | ✓   | ✓   | ✓   | ✓   | ✓  | ✓        | —       | —    |
|               | "serializable"               | serializable       | ✓  | ✓   | ✓   | ✓   | ✓   | ✓   | ✓  | ✓        | ✓       | ✓    |
|:--------------|:---------------------------  |:-------------------|:--:|:---:|:---:|:---:|:---:|:---:|:--:|:--------:|:-------:|:----:|
| MySQL/InnoDB  | "read uncommitted"           | read uncommitted   | ✓  | —   | —   | —   | —   | —   | —  | —        | —       | —    |
|               | "read committed"             | MAV                | ✓  | ✓   | ✓   | ✓   | ✓   | —   | —  | —        | —       | —    |
|               | "repeatable read" ★          | MAV                | ✓  | ✓   | ✓   | ✓   | ✓   | R/O | —  | R/O      | —       | —    |
|               | "serializable"               | serializable       | ✓  | ✓   | ✓   | ✓   | ✓   | ✓   | ✓  | ✓        | ✓       | ✓    |
|:--------------|:---------------------------  |:-------------------|:--:|:---:|:---:|:---:|:---:|:---:|:--:|:--------:|:-------:|:----:|
| Oracle DB     | "read committed" ★           | MAV                | ✓  | ✓   | ✓   | ✓   | ✓   | —   | —  | —        | —       | —    |
|               | "serializable"               | snapshot isolation | ✓  | ✓   | ✓   | ✓   | ✓   | ✓   | ✓  | ✓        | —       | some |
|:--------------|:---------------------------  |:-------------------|:--:|:---:|:---:|:---:|:---:|:---:|:--:|:--------:|:-------:|:----:|
| MS SQL Server | "read uncommitted"           | read uncommitted   | ✓  | —   | —   | —   | —   | —   | —  | —        | —       | —    |
|               | "read committed" (locking) ★ | MAV                | ✓  | ✓   | ✓   | ✓   | ✓   | —   | —  | —        | —       | —    |
|               | "read committed" (snapshot)  | MAV                | ✓  | ✓   | ✓   | ✓   | ✓   | —   | —  | —        | —       | —    |
|               | "repeatable read"            | repeatable read    | ✓  | ✓   | ✓   | ✓   | ✓   | —   | ✓  | some     | ✓       | —    |
|               | "snapshot"                   | snapshot isolation | ✓  | ✓   | ✓   | ✓   | ✓   | ✓   | ✓  | ✓        | —       | —    |
|               | "serializable"               | serializable       | ✓  | ✓   | ✓   | ✓   | ✓   | ✓   | ✓  | ✓        | ✓       | ✓    |
|:--------------|:---------------------------  |:-------------------|:--:|:---:|:---:|:---:|:---:|:---:|:--:|:--------:|:-------:|:----:|

Legend:

* ★ = default configuration
* ✓ = isolation level prevents this anomaly from occurring
* — = isolation level does not prevent this anomaly, so it can occur
* R/O = isolation level prevents this anomaly in a read-only context, but when you perform writes,
  the anomaly can occur (see test cases for details)
* some = isolation level prevents this anomaly in some cases, but not in others (see test cases for details)

Vendor documentation of isolation levels:

* [PostgreSQL](http://www.postgresql.org/docs/9.3/static/transaction-iso.html)
* [MySQL/InnoDB](http://dev.mysql.com/doc/refman/5.7/en/set-transaction.html)
* [Oracle](https://docs.oracle.com/cd/B28359_01/server.111/b28318/consist.htm)
* [SQL Server](http://msdn.microsoft.com/en-us/library/ms173763.aspx)


* ANSI Fuzzy Read (P2) is a degenerate case of Read Skew
* Read Skew = Berenson et al's A5A = Adya's G-single
  (actually "G-item-single"? preventing G-single also prevents phantoms)
* Write Skew = Berenson et al's A5B = Adya's G2-item
