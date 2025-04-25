+++
categories = ['databases', 'postgresql', 'performance']
date = '2025-04-20T10:53:36+01:00'
draft = false
tags = ['postgresql', 'performance', 'bloat', 'toast']
title = 'Query Performance Improvement Part 2'
+++

# Overview

[Part 1](/post/query-performance-improvement-part-1) looks at some principles and introduces some of the tools in your arsenal.

Part 2 (this post) covers the dreaded PostgreSQL bloat issue as well as how TOAST can help performance, or hinder it.

[Part 3](/post/query-performance-improvement-part-3) dives deeper into identifying, investigating and mitigating slow queries.

[Part 4](/post/query-performance-improvement-part-4) covers indexing.

[Part 5](/post/query-performance-improvement-part-5) covers rewriting queries.

# TL;DR

- Bloat occurs because PostgreSQL does not update in place, it creates a new row and marks the old one as obsolete
- VACUUM fixes bloat, well, it marks obsolete rows as re-usable space
- VACUUM FULL fixes bloat by compactly rewriting the data file and rebuilding indexes
- You should never run VACUUM FULL because it requires a full table lock
- pg_repack is an option ... possibly
- Use pg_repack with care - it can have problematic side effects
- TOAST compresses wide columns so that the resulting row is < 2kB
- If the table is still too wide, then compressed wide columns are split into 2kB chunks and move to a TOAST table
- A 2kB (ish) wide table can be problematic

# Bloat

## What is bloat?

Strictly speaking, bloat is the unused space left behind by a VACUUM that is not a VACUUM FULL.

However, in common use, it means any space occupied as a result of PostgreSQL's UPDATE mechanism. PostgreSQL, unlike some other databases does not update data in place, it writes a new row (with all the index consequences of that!) and marks the old row as inactive. This allows PostgreSQL's multi-version concurrency control (MVCC) to operate.

What this means is that although you might think you only have one row in a table for a given primary key, there could actually be dozens or even hundreds of inactive copies taking up space, which not only potentially costs money, but also impacts performance because index traversal occurs on bigger index files and data reads occur on bigger data files.

## Types of bloat?

The most commonly considered forms of bloat are table bloat and index bloat. There is also catalog bloat, which is the special case of table and index bloat in the PostgreSQL catalog tables.

## How do you fix bloat?

The VACUUM process fixes table bloat ... sort of. What it does is to mark the inactive rows as free space that the engine can reuse. This free space is the strict definition of bloat.

To get rid of table bloat AND index bloat, you need to run VACUUM FULL. VACUUM FULL rewrites the table and indexes to new files, eliminating any bloat.

The only problem is that you should never run VACUUM FULL, because it requires a table lock for the duration of the rewrite, which could be several minutes on a very large table and is best done during an arranged outage.

Another way to address index bloat is to concurrently create a duplicate of the bloated index, drop the bloated index and rename the duplicate to the formerly bloated index. Since the index drop is usually instantaneous, I don't bother with dropping the index concurrently, but if you are concerned about this for any reason then you can DROP INDEX CONCURRENTLY.

So, assuming table t1 with index i1 on column c1:

> CREATE INDEX CONCURRENTLY i1_new ON t1(c1);
> DROP INDEX i1;
> ALTER INDEX i1_new RENAME TO i1;

## What is so special about catalog bloat?

Bloat of an application table tends to cause issues with the application while addressing that table, which makes it much more specific, visible, and identifiable. Catalog bloat tends to present itself much more vaguely and potentially with a much bigger blast radius. It gets fixed in the same way, though.

An example of this could be a system that creates and destroys temporary tables very frequently. This can lead to pg_class and pg_attributes being huge if these tables are not VACUUMed frequently enough.

## pg_repack

[pg_repack](https://reorg.github.io/pg_repack/) is a PostgreSQL extension which lets you remove bloat from tables and indexes, and optionally restore the physical order of clustered indexes. Unlike CLUSTER and VACUUM FULL it works online, without holding an exclusive lock on the processed tables during processing.

This might sound like a perfect alternative to VACUUM FULL, but Percona have done a write up of what can go wrong and how to avoid it [here](https://www.percona.com/blog/understanding-pg_repack-what-can-go-wrong-and-how-to-avoid-it/). This is another case of There Ain't No Such Thing As A Free Lunch.

# TOAST

## What is TOAST?

The Oversized-Attribute Storage Technique is described in the [PostgreSQL documentation](https://www.postgresql.org/docs/current/storage-toast.html) as follows:

> PostgreSQL uses a fixed page size (commonly 8 kB), and does not allow tuples to span multiple pages. Therefore, it is not possible to store very large field values directly. To overcome this limitation, large field values are compressed and/or broken up into multiple physical rows. This happens transparently to the user, with only small impact on most of the backend code. The technique is affectionately known as TOAST (or “the best thing since sliced bread”). The TOAST infrastructure is also used to improve handling of large data values in-memory.

## How does TOAST help?

If a row is wider than TOAST_TUPLE_THRESHOLD (default 2kB), and columns will benefit from compression, then TOAST compresses wide columns. If the row is still too wide after compression, TOAST splits the wide columns into chunks of roughly 2000 bytes and moves the chunked data into a TOAST table. I call this "full TOASTing". This keeps the main table small, improving cacheing and, if the TOASTed data is not changed in an update, then no movement of data is required for the TOASTed data, improving performance and reducing bloat.

If the row is within TOAST_TUPLE_THRESHOLD after compression then the compressed data is NOT moved to a TOAST table but remains inline.

## What can go wrong?

If the row is narrow, TOAST is unlikely to kick in and so is irrelevant. If the table is wide and the TOASTed columns are moved into a TOAST table, then TOAST delivers all possible advantages.

However, then is an edge case between these two extremes: if the table is wide, but the TOASTed columns are not wide enough to be moved into a TOAST table, then the row is potentially large enough to lead to performance issues. Haki Benita examines this very thoroughly [here](https://hakibenita.com/sql-medium-text-performance).

Two potential fixes for this are to use a smaller `toast_tuple_target` for your table, or move the offending column to a separate table.

Another potential problem is if you update a TOASTed column, all the pain of bloat occurs and it can be on much bigger tables than the main table.

# Summary

In this post we looked at:
- what bloat is and how to fix it
- what TOAST is, how it helps performance and how it can fail to help performance

---

Previous: [Fundamentals](/post/query-performance-improvement-part-1/)     |     Next: [Identifying, investigating and mitigating](/post/query-performance-improvement-part-3)
