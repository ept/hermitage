Hermitage: Testing transaction isolation levels
===============================================

> “Aristotle maintained that women have fewer teeth than men; although he was twice married, it
> never occurred to him to verify this statement by examining his wives' mouths.”
>
> ― Bertrand Russell, The Impact of Science on Society (1952)

[Hermitage](https://github.com/ept/hermitage) is an attempt to nail down precisely what
different database systems actually mean with their isolation levels. It's a suite of tests that
simulates various concurrency issues — some common, some more obscure — and documents how different
databases handle those situations.

This project was started by [Martin Kleppmann](http://martin.kleppmann.com/) as background research
for his book, [Designing Data-Intensive Applications](http://dataintensive.net/). In this repository
you'll find a lot of nitty-gritty detail. For a gentle, friendly introduction to the topic, please
read the book. There is also a
[blog post](http://martin.kleppmann.com/2014/11/25/hermitage-testing-the-i-in-acid.html)
with some background story.


Summary of test results
-----------------------

The cryptic abbreviations (G1c, PMP etc) are different kinds of concurrency *anomalies* — issues
which can occur when multiple clients are executing transactions at the same time, and which can
cause application bugs. The precise definitions of these anomalies are given in the literature
(see below for details).

| DBMS          | So-called isolation level    | Actual isolation level | G0 | G1a | G1b | G1c | OTV | PMP | P4 | G-single | G2-item | G2   |
|:--------------|:-----------------------------|:-----------------------|:--:|:---:|:---:|:---:|:---:|:---:|:--:|:--------:|:-------:|:----:|
| PostgreSQL    | "read committed" ★           | monotonic atomic view | ✓  | ✓   | ✓   | ✓   | ✓   | —   | —  | —        | —       | —    |
|               | "repeatable read"            | snapshot isolation     | ✓  | ✓   | ✓   | ✓   | ✓   | ✓   | ✓  | ✓        | —       | —    |
|               | "serializable"               | serializable           | ✓  | ✓   | ✓   | ✓   | ✓   | ✓   | ✓  | ✓        | ✓       | ✓    |
|               |                              |                        |    |     |     |     |     |     |    |          |         |      |
| MySQL/InnoDB  | "read uncommitted"           | read uncommitted       | ✓  | —   | —   | —   | —   | —   | —  | —        | —       | —    |
|               | "read committed"             | monotonic atomic view | ✓  | ✓   | ✓   | ✓   | ✓   | —   | —  | —        | —       | —    |
|               | "repeatable read" ★          | monotonic atomic view | ✓  | ✓   | ✓   | ✓   | ✓   | R/O | —  | R/O      | —       | —    |
|               | "serializable"               | serializable           | ✓  | ✓   | ✓   | ✓   | ✓   | ✓   | ✓  | ✓        | ✓       | ✓    |
|               |                              |                        |    |     |     |     |     |     |    |          |         |      |
| Oracle DB     | "read committed" ★           | monotonic atomic view | ✓  | ✓   | ✓   | ✓   | ✓   | —   | —  | —        | —       | —    |
|               | "serializable"               | snapshot isolation     | ✓  | ✓   | ✓   | ✓   | ✓   | ✓   | ✓  | ✓        | —       | some |
|               |                              |                        |    |     |     |     |     |     |    |          |         |      |
| MS SQL Server | "read uncommitted"           | read uncommitted       | ✓  | —   | —   | —   | —   | —   | —  | —        | —       | —    |
|               | "read committed" (locking) ★ | monotonic atomic view | ✓  | ✓   | ✓   | ✓   | ✓   | —   | —  | —        | —       | —    |
|               | "read committed" (snapshot)  | monotonic atomic view | ✓  | ✓   | ✓   | ✓   | ✓   | —   | —  | —        | —       | —    |
|               | "repeatable read"            | repeatable read        | ✓  | ✓   | ✓   | ✓   | ✓   | —   | ✓  | some     | ✓       | —    |
|               | "snapshot"                   | snapshot isolation     | ✓  | ✓   | ✓   | ✓   | ✓   | ✓   | ✓  | ✓        | —       | —    |
|               | "serializable"               | serializable           | ✓  | ✓   | ✓   | ✓   | ✓   | ✓   | ✓  | ✓        | ✓       | ✓    |
|               |                              |                        |    |     |     |     |     |     |    |          |         |      |
| FDB SQL Layer | "serializable" ★             | serializable           | ✓  | ✓   | ✓   | ✓   | ✓   | ✓   | ✓  | ✓        | ✓       | ✓    |

Legend:

* ★ = default configuration
* ✓ = isolation level prevents this anomaly from occurring
* — = isolation level does not prevent this anomaly, so it can occur
* R/O = isolation level prevents this anomaly in a read-only context, but when you perform writes,
  the anomaly can occur (see test cases for details)
* some = isolation level prevents this anomaly in some cases, but not in others (see test cases for details)
* anomalies
  - G0: Write Cycles, Dirty Writes
  - G1a: Aborted Reads
  - G1b: Intermediate Reads
  - G1c: Circular Information Flow
  - OTV: Observed Transaction Vanishes
  - PMP: Predicate-Many-Preceders
  - P4: Lost Update
  - G-single: Read Skew, Single Anti-dependency Cycles
  - G2-item: Write Skew, Item Anti-dependency Cycles
  - G2: Anti-Dependency Cycles


Background
----------

*Isolation* is the I in ACID, and it describes how a database protects an application from
concurrency problems (race conditions). If you read a traditional 
[database theory textbook](http://research.microsoft.com/en-us/people/philbe/ccontrol.aspx),
it will tell you that isolation is supposed to mean *serializability*, i.e. you can pretend
that transactions are executed one after another, and concurrency problems do not happen.
However, if you look at the implementations of
[isolation in practice](http://www.bailis.org/blog/when-is-acid-acid-rarely/), you see that
serializability is rarely used, and some popular databases (such as Oracle) don't even implement it.

So what does isolation actually mean? Well, in practice, many database systems allow you to choose your
isolation level, as a trade-off between performance and safety (weaker isolation is faster but exposes
you to more potential race conditions). Unfortunately, those weaker isolation levels are quite
[poorly understood](http://www.bailis.org/blog/understanding-weak-isolation-is-a-serious-problem/).
Even though our industry has been working with this stuff for 20 years or more, there are not many
people who can explain off-the-cuff the difference between, say, *read committed* and *repeatable read*.
This is a problem, because if you don't know what guarantees you can expect from your database, you
cannot know whether your code has concurrency bugs and race conditions.

The [SQL standard](http://synthesis.ipi.ac.ru/synthesis/student/oodb/essayRef/sqlFoundation) tried
to define four isolation levels (read uncommitted, read committed, repeatable read and serializable),
but its definition is [flawed](http://research.microsoft.com/pubs/69541/tr-95-51.pdf). Several
researchers have tried to nail down more precise definitions of weak (i.e. non-serializable) isolation
levels. In particular:

* Peter Bailis, Aaron Davidson, Alan Fekete, Ali Ghodsi, Joseph M Hellerstein and Ion Stoica:
  “[Highly Available Transactions: Virtues and Limitations (Extended Version)](http://arxiv.org/pdf/1302.0309.pdf),”
  at *40th International Conference on Very Large Data Bases* (VLDB), September 2014.
* Alan Fekete, Dimitrios Liarokapis, Elizabeth O'Neil, Patrick O'Neil, and Dennis Shasha:
  “[Making Snapshot Isolation Serializable](http://www.researchgate.net/publication/220225203_Making_snapshot_isolation_serializable/file/e0b49520567eace81f.pdf),”
  *ACM Transactions on Database Systems* (TODS), volume 30, number 2, pages 492–528, June 2005.
  [doi:10.1145/1071610.1071615](http://dx.doi.org/10.1145/1071610.1071615)
* Atul Adya: “[Weak Consistency: A Generalized Theory and Optimistic Implementations for Distributed
  Transactions](http://pmg.csail.mit.edu/papers/adya-phd.pdf),” PhD Thesis, Massachusetts Institute of
  Technology, Cambridge, MA, USA, March 1999.
* Hal Berenson, Phil Bernstein, Jim Gray, Jim Melton, Elizabeth O'Neil and Patrick O'Neil:
  “[A Critique of ANSI SQL Isolation Levels](http://research.microsoft.com/pubs/69541/tr-95-51.pdf),”
  at *ACM International Conference on Management of Data* (SIGMOD), volume 24, number 2, May 1995.
  [doi:10.1145/568271.223785](http://dx.doi.org/10.1145/568271.223785)
* Jim N Gray, Raymond A Lorie, Gianfranco R Putzolu, and Irving L Traiger:
  “[Granularity of Locks and Degrees of Consistency in a Shared Data
  Base](http://citeseer.ist.psu.edu/viewdoc/download?doi=10.1.1.92.8248&rep=rep1&type=pdf),”
  in *Modelling in Data Base Management Systems: Proceedings of the IFIP Working Conference on
  Modelling in Data Base Management Systems*, edited by G.M. Nijssen, Elsevier/North Holland
  Publishing, pages 364–394, 1976.
  Also in [*Readings in Database Systems*](http://redbook.cs.berkeley.edu/), edited by Joseph M.
  Hellerstein and Michael Stonebraker, 4th edition, MIT Press, 2005. ISBN: 978-0-262-69314-1

This project is based on the formal definition of weak isolation introduced by Adya, as extended by
Bailis et al. They mathematically define certain *anomalies* (or *phenomena*) which can occur in an
unrestricted concurrency model, and define isolation levels as *prohibiting* or *preventing* certain
anomalies from occurring.

The formal definitions are not easy to understand, but at least they are precise. By comparison, the
database vendors' documentation of isolation levels is also hard to understand, but on top of that
it's also frustratingly vague:

* [PostgreSQL](http://www.postgresql.org/docs/current/static/transaction-iso.html)
* [MySQL/InnoDB](http://dev.mysql.com/doc/refman/5.7/en/set-transaction.html)
* [Oracle](https://docs.oracle.com/cd/B28359_01/server.111/b28318/consist.htm)
* [SQL Server](http://msdn.microsoft.com/en-us/library/ms173763.aspx)
* [FoundationDB](https://foundationdb.com/key-value-store/documentation/developer-guide.html#transactions-in-foundationdb)

Goals of this project
---------------------

This repository contains a series of tests which probe for a range of concurrency anomalies.
They are based on the definitions in the literature above. This is useful for several reasons:

* It allows us to compare isolation levels easily: the more check marks in the table above,
  the stronger its guarantees.
* For anyone who needs help choosing the right isolation level for their application, the test
  suites provide concrete examples of the differences between isolation levels.
* Various new databases have [claimed](https://foundationdb.com/acid-claims) to support
  ACID transactions, but their marketing materials often don't make clear what guarantees are
  actually provided. This test suite can allow a fair comparison of different databases, at least
  on the isolation aspect of ACID.
* Hopefully, this effort can be part of a journey towards a better understanding of weak
  isolation. It looks like weak isolation isn't going away, so we need to learn to be more
  precise about what it means, and build tools to help us deal with it, otherwise we'll just
  continue creating buggy applications.



Caveats
-------

* This is a test suite. It obviously cannot prove that a database always behaves in a certain way,
  it can only probe certain examples and observe what happens.
* Tests are currently executed by hand. This means that any concurrency issues that depend on fast
  timings will not be found. However, it's remarkable that even at the slow speed of a human, you
  can still easily demonstrate concurrency issues. It's not the speed that matters, it's the
  ordering of events.
* The summary table above only describes safety properties, i.e. whether the database allows a
  certain race condition to occur. It doesn't describe how the anomaly is prevented (usually by
  blocking or aborting some of the transactions). In practice, how much transactions need to be
  blocked or aborted makes a big performance difference. For example, although PostgreSQL's
  serializable and MySQL's serializable have the same isolation guarantees, they have
  [totally different implementations](http://drkp.net/papers/ssi-vldb12.pdf) and very different
  performance characteristics.
* We're not trying to compare performance here. Performance depends on the workload, so please
  do your own benchmarking.
* More check marks doesn't necessarily mean better. This is not
  [Top Trumps](http://en.wikipedia.org/wiki/Top_Trumps), it's a game of trade-offs. All we're
  trying to do here is to understand what we gain and what we lose at different isolation levels.


Using this project
------------------

The tests are currently executed by hand: you simply open two or three connections to the
same database in different terminal windows, and run the queries in the order they appear
in the test script. A comment indicates which transaction executes a particular query, and
what the expected result is.

This could probably be automated, but it's actually quite interesting to go through the
exercise of stepping through transactions one line at a time, and watching how the
database responds. If you want to build an intuition for database concurrency, running
through the test suite is a good exercise. For some databases, setup instructions are
included at the bottom of the file.

At the moment, this project only compares five databases, but many more databases offer
transactions. It would be especially interesting to add the new generation of distributed
transactional databases ("NewSQL" if you like marketing-speak) to this comparison:
Aerospike, NuoDB, MemSQL, etc. FoundationDB is currently included.

If you would like to port the test suite to another database, or add new tests, your
contribution would be most welcome!
 
Thank you to contributors:

* [Jennifer Rullmann](https://twitter.com/jrullmann) for porting the test suite to FoundationDB.


License
-------

Copyright Martin Kleppmann, 2014. This work is licensed under a
[Creative Commons Attribution 4.0 International License](http://creativecommons.org/licenses/by/4.0/).

[![Creative Commons License](https://i.creativecommons.org/l/by/4.0/88x31.png)](http://creativecommons.org/licenses/by/4.0/)
