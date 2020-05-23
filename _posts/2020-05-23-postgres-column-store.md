---
title: "Waiting for a column store in Postgres"
date: 2020-05-23T07:03:53+02:00
---

Databases have been popular for storing and retrieval of information for decades and they have offered ways to work with all sorts of shapes and sizes of data and without specialising in any kind of retrieval or functionality. Over the past decade, there has been an emergence of a _analytical_ databases, which leverage the fact that some data, chiefly in reporting and analytics, are written once and read many times. Updates are infrequent, so are individual row lookups. How are these engines different from what we have in Postgres or MySQL? There are a few concepts to unfold.

### Rows and columns
In order to support heterogenous workloads and strict guarantees ([ACID](https://en.wikipedia.org/wiki/ACID), [MVCC](https://en.wikipedia.org/wiki/Multiversion_concurrency_control) and others), databases typically store all the data for a given row in one chunk. A table of names and ages would, in a grossly simplified manner, be stored as `joe,20;jane,24;doug,30;lynn,21`, which works alright, but the "issue" is that when we want to query for the average age of all the people in our contacts list, we need to read all the names as well. This is not an issue in this table, but imagine wide tables with a 100 columns (no uncommon in a production setting). You suddenly read a lot more data than you need.

One mitigation is to use indexes. They work great, but they pose a few problems in analytics - 1) You ~duplicate storage, 2) all the table operations are more expensive, because you need to keep indexes in sync, 3) ideally you'd have an index for each column, because you often don't know what columns you'll query, thus amplifying issues one and two.

This is where column store come in. Instead of storing data by rows, they keep column information together, so our example above would yield `joe,jane,doug,lynn;20,24,30,21`, so a question for an average age or whether or not we know any Dougs would be much faster than in a row store. In contrast, looking up all the information for Doug would be fairly slow in a column store, because you need to look in `n` places for `n` columns. You know, tradeoffs.

### Engines and storage

While we think of databases as homogenous, one stop shops, but they have a few more or less modular pieces. The critical piece here is the storage system - e.g. MySQL has historically had storage engines swappable, that's why it has seen a number of implementations over the years.

The way Postgres users got around it varied. Either new extensions got built - think [Citus](https://github.com/citusdata/citus) or [TimescaleDB](https://github.com/timescale/timescaledb) - these allow for somewhat native improvements in functionality, but they are not shipped with Postgres, they are not first class citizens, they are often not supported in hosted solutions (like [Amazon RDS](https://aws.amazon.com/rds/)) and they need to keep pace with upstream Postgres to keep being compatible. The other option is a hard fork, one of the more notable is [Amazon Redshift](https://aws.amazon.com/redshift/), Amazon's data warehousing solution, which started as a fork of Postgres 8 and has lived its own life - new Postgres features didn't get incorporated and, more importantly, the whole design of this columnar store sacrificed some functionality surrounding integrity and table design (e.g. you can't change column types on the fly, it's as frustrating as it sounds).

Postgres has historically taken this different path from MySQL, it's always had the Postgres storage system as the only way of maintaining data, that is until last spring (2019), when [pluggable storage got committed](https://commitfest.postgresql.org/19/1283/).

This change now has sparked a new interest in new storage engines, e.g. [zheap](https://github.com/EnterpriseDB/zheap). But the one I'd like to briefly talk about is [zedstore](https://github.com/greenplum-db/postgres/tree/zedstore).

Zedstore finally comes as a first class implementation of a column store with all the guarantees and APIs Postgres currently offers, be it MVCC, ACID, transactional DDL etc. If you like running or testing databases locally, I encourage you to try this one. I've never built Postgres myself, but it turned out to be pretty simple.

First clone the repo linked above, configure it using the recommended option noted in the README and then follow the [upstream installation docs](https://www.postgresql.org/docs/current/install-procedure.html). If you're already running Postgres, make sure to edit the port used by this zedstore branch (I used 5433). Then just create a database, initialise it and run your new build against this new database on a new port.

I tested zedstore against a real workload I was running at the time, it was quite a full table scan heavy set of queries, an ideal workload for a column store. I got 3-10x speedups for most aggregation and filter queries. In some cases, I got slowdowns, this was especially notable when running expensive calculations like `count(distinct)`.

### Conclusion

The main advantage of having a column store built into Postgres is that one can easily run hybrid workloads, where you have row-based OLTP-like tables for your day to day operations and then some column store tables/views for analytics and reporting. This approach saves you the hassle of synchronising data between databases, which has been a pain we've had to endure for too long. It's something Microsoft's SQL Server has offered from quite some time and I'm glad open source databases are catching up.

I think the potential of removing ETL processes is the main benefit of all this, but there are some other considerations Ihaven't talked about - notably space savings thanks to on-disk compression, among other things.

