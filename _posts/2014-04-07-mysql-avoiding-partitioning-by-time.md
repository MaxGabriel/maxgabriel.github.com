---
layout: post
title: "MySQL – Avoiding Partitioning a Table by Time"
---

> __*Warning*__: This blog post is speculative. I haven't tested this approach to confirm that it works or ran benchmarks. Don't use this in production without verifying it works.

When using a MySQL table to store huge amounts of time-sensitive data (perhaps you're tracking shipments and you only need the data going back one month), one approach is to partition the table by time. If your queries are limited by a time range then you'll only have to search a few partitions, speeding up queries. With this approach you don't need to delete old data (in an attempt to keep the table small), and if you do, deleting it will be very efficient.

Unfortunately, this partitioning strategy carries a cost:

> Any primary key or unique index must include all columns in the partitioning expression. 

> *–High Performance MySQL 3rd Edition, p. 266*

In other words, partitioning on time precludes you from using an auto-incrementing integer primary key. Without a primary key to enforce uniqueness, you might fallback to something like a UUID string to uniquely identify rows. But this has two hidden costs, and they're rooted in the way that InnoDB works. 

The first problem is write contention. InnoDB stores a row's values inside a balanced tree where the primary key is kept—this is called a clustered index. Without a true primary key or unique index, MySQL will [fallback to using a hidden integer primary key](http://dev.mysql.com/doc/refman/5.0/en/innodb-index-types.html) to create this clustered index. Under the hood, this hidden integer key is [globally shared between tables](http://blog.jcole.us/2013/05/02/how-does-innodb-behave-without-a-primary-key/), which may create write contention as multiple tables try to get the next primary key value.  

The second problem is slower reads. Because all the values of a row are kept in the clustered index, looking up a row by a secondary index (like the `UUID` column) requires finding the row in the secondary index, grabbing the hidden primary key from there, and then using the hidden primary key to lookup the row's values in the clustered index (at least doubling the amount of searching to find a particular row).

This leaves us an impasse—partitioning every N rows on an auto-incrementing primary key gives us a real primary key to work with, but then queries *not* using the primary key (but searching within e.g. the last two weeks) might become too slow.

#### A workaround

A workaround I came up with is to use the auto-incrementing nature of the primary key as a proxy for when a row was created. If you find the primary key of the first shipment created >3 weeks ago, you know that any row with a primary key greater than that occurred within the last 3 weeks. You can run a regularly scheduled cron job to take update this value.<sup>1</sup> Then, you would rewrite queries like:

```sql
SELECT * FROM shipments WHERE customer_id = 7 AND (created_at BETWEEN '2014-04-07 05:16:51' AND '2014-04-08 05:16:51');
```

as

```sql
SELECT * FROM shipments WHERE customer_id = 7 AND (created_at BETWEEN '2014-04-07 05:16:51' AND '2014-04-08 05:16:51') AND id > 4520000;
```

The second query will use the partitioning function to limit the partitions it searches. From there, an index on `customer_id` would get you the rest of the way.

<hr>

<sup>1</sup> You could also keep a set of values; e.g. this primary key was created 3 weeks ago, this one was created 2 weeks ago, etc.
