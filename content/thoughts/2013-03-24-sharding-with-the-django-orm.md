---
title: "Sharding with the Django ORM"
created_at: 2013-03-24 11:03:00 -0700
kind: article
---

Recently, I gave a talk at the first PyPgDay on how Disqus shards data with
the ORM. PyPgDay was the first of its kind, an action-packed summit for
python developers and postgres fanatics to share their knowledge on both
topics. It was my first foray into presenting at a technical conference - the
opportunity was perfect, given the size of the audience and typical background
of the attendees. With Pycon moving to Montreal next year, I hope Josh and the
team at [PG Experts](http://www.pgexperts.com/) continue building on the early
success of PyPgDay next year.

[The slides are available on Speaker Deck.](https://speakerdeck.com/mikeclarke)

Sharding is a horizontal partitioning technique that splits rows of a database
table across multiple database instances. Many applications will never need to
concern themselves with this class of scaling concerns; databases are very
efficient at shuffling data between memory and disk, and the read / write load
can be managed with a single host. For applications that scale to millions of
users or otherwise outgrow the performance a single machine offers, sharding is
a great answer.

There are many benefits of sharding, specifically:

* smaller database instances
* faster queries
* cheaper hardware
* easier maintenance
* horizontal scalability

With these advantages, why would a web application chose not to shard their
data? The challenges of sharding are many, including:

* increased complexity
* diminished reliability
* less flexibility in querying data

Given the tradeoffs, most applications wisely choose to begin with a single
database instance and worry about sharding later. "Later" is now at Disqus,
given the size of its network, so we have been carefully moving data over
time. Historically, Disqus has scaled the database tier vertically, buying
increasingly more powerful hardware and shuffling read load to replicated
slaves. This carries its own challenges that sharding solves.

Sharding is not easy, particularly after an application has reached a certain
size. To buy time, Disqus has used several techniques to allow us to control
database utilization:

* Throttling writes by deferring them via celery / RabbitMQ
* Moving SELECTs to read slaves
* Isolating databases based on query patterns (roles)

Several strategies can be employed when sharding Postgres databases. At Disqus,
we chose to shard based on table name. Where as the original table name as
generated by Django might be comments_post, our sharding tools will rewrite
the SQL to query a table comments_post_X, where X is the shard ID calculated
based on a consistent hashing scheme. All these tables live in a single schema,
on a single database instance. We chose this approach because it worked best
with our logical replication tool, [Slony](http://slony.info/). Using
table-based replication, we simplified the data migration.

Instgram, on the other hand, shards via schemaname. Other companies have opted
shard based on database name. These approaches simplify some aspects of
sharding data, because they avoid rewriting SQL and sharding occurs at the
connection level. Your mileage may vary.

As we identify tables to shard, we tend to focus on two access patterns. The
first is high throughput tables, in terms of INSERTs and UPDATEs. These tables
tend to place the highest demands on disk I/O. For Disqus, this data includes
new comments, votes, and threads (a Disqus thread is the comment tree on each
unique page). Data that does not change as often - such as new user and site
signups - is less important to shard.

The second pattern that we tend to shard is very large tables that may not be
accessed as frequently as other data sets. Large tables with fat rows carry
extra overhead in terms of straining replication, bloating the size of the
database, and wasting precious disk iops. At Disqus, we track various metadata
about comments and votes in the database. This data grows quickly, but it is
not queried nearly as requently as other data. This makes it an ideal candidate
to shard.

One of the biggest challenges of sharding is increased complexity for
developers in local development. What if there was a way to use the Django ORM
in a way that feels familiar to devs but under the hood it routes queries to the
proper database backend? The infrastructure team at Disqus has created this
magic, such that our developers can continue to hack on features while
blisfully unaware of the underlying implementation details (mostly).

[The sample sharding application](https://github.com/disqus/sharding-example)
includes tools similar to the ones used in production at Disqus today. Notably,
there are 4 tools included in the example:

* Database connection config helper for use in settings.py
* Field for generating unique, sortable primary keys across shards
* DB query router based on shard ids
* A management command `sqlpartition` for generating DDL

The provided example app demonstrates how the sqlshards tooling might be used.
The project is a bit rough, but with proper test coverage and better docs, the
app could be useful beyond serving as an example of what we do at Disqus.