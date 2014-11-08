Transaction isolation level test suite
======================================

TODO description.

| DBMS          | So-called isolation level   | Actual isolation   | G0 | G1a | G1b | G1c | OTV | PMP | P4 | G-single | G2-item | G2   |
|:--------------|:----------------------------|:-------------------|:--:|:---:|:---:|:---:|:---:|:---:|:--:|:--------:|:-------:|:----:|
| PostgreSQL    | "read committed"            | MAV                | ✓  | ✓   | ✓   | ✓   | ✓   | —   | —  | —        | —       | —    |
| PostgreSQL    | "repeatable read"           | snapshot isolation | ✓  | ✓   | ✓   | ✓   | ✓   | ✓   | ✓  | ✓        | —       | —    |
| PostgreSQL    | "serializable"              | serializable       | ✓  | ✓   | ✓   | ✓   | ✓   | ✓   | ✓  | ✓        | ✓       | ✓    |
| MySQL/InnoDB  | "read uncommitted"          | read uncommitted   | ✓  | —   | —   | —   | —   | —   | —  | —        | —       | —    |
| MySQL/InnoDB  | "read committed"            | MAV                | ✓  | ✓   | ✓   | ✓   | ✓   | —   | —  | —        | —       | —    |
| MySQL/InnoDB  | "repeatable read"           | MAV                | ✓  | ✓   | ✓   | ✓   | ✓   | R/O | —  | R/O      | —       | —    |
| MySQL/InnoDB  | "serializable"              | serializable       | ✓  | ✓   | ✓   | ✓   | ✓   | ✓   | ✓  | ✓        | ✓       | ✓    |
| Oracle DB     | "read committed"            | MAV                | ✓  | ✓   | ✓   | ✓   | ✓   | —   | —  | —        | —       | —    |
| Oracle DB     | "serializable"              | snapshot isolation | ✓  | ✓   | ✓   | ✓   | ✓   | ✓   | ✓  | ✓        | —       | some |
| MS SQL Server | "read uncommitted"          | read uncommitted   | ✓  | —   | —   | —   | —   | —   | —  | —        | —       | —    |
| MS SQL Server | "read committed" (locking)  | MAV                | ✓  | ✓   | ✓   | ✓   | ✓   | —   | —  | —        | —       | —    |
| MS SQL Server | "read committed" (snapshot) | MAV                | ✓  | ✓   | ✓   | ✓   | ✓   | —   | —  | —        | —       | —    |
| MS SQL Server | "repeatable read"           | repeatable read    | ✓  | ✓   | ✓   | ✓   | ✓   | —   | ✓  | some     | ✓       | —    |
| MS SQL Server | "snapshot"                  | snapshot isolation | ✓  | ✓   | ✓   | ✓   | ✓   | ✓   | ✓  | ✓        | —       | —    |
| MS SQL Server | "serializable"              | serializable       | ✓  | ✓   | ✓   | ✓   | ✓   | ✓   | ✓  | ✓        | ✓       | ✓    |


* ANSI Fuzzy Read (P2) is a degenerate case of Read Skew
* Read Skew = Berenson et al's A5A = Adya's G-single
  (actually "G-item-single"? preventing G-single also prevents phantoms)
* Write Skew = Berenson et al's A5B = Adya's G2-item
