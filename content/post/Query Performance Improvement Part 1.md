+++
categories = ['databases', 'postgresql', 'performance']
date = '2025-04-19T01:30:19+01:00'
draft = false
tags = ['postgresql', 'performance', 'serializable', 'analyze']
title = 'Query Performance Improvement Part 1'
+++

# Overview

Part 1 (this post) looks at some principles and introduces some of the tools in your arsenal.

[Part 2](/post/query-performance-improvement-part-2) covers the dreaded PostgreSQL bloat issue as well as how TOAST can help performance, or not.

[Part 3](/post/query-performance-improvement-part-3) dives deeper into identifying, investigating and mitigating slow queries.

[Part 4](/post/query-performance-improvement-part-4) covers indexing.

[Part 5](/post/query-performance-improvement-part-5) covers rewriting queries.

# TL;DR

- Scaling up or out is more expensive than spending a couple of hours optimising
- Get answers, not data
- Lazy load lookups
- Eliminate non-essential queries
- Don't use SERIALIZABLE isolation
- Make sure auto-analyze is running and have a manual ANALYZE schedule as a safety net
- Denormalising is usually the wrong solution to performance issues

# Introduction

Welcome to this series of blog posts which will hopefully walk you through everything you need to know about improving query performance, specifically in PostgreSQL. The basic principles apply to all databases, though, even if implementation differences mean that different databases find different things painful.

## Who is this for?

It's for anyone who works with databases, or is about to work with databases and wants to know how to make the best use of the power available: DBAs, developers, system admins, engineers as well as non-technical folk who may be curious about this mysterious process.

## Caveats

I've tried to make this as comprehensive as possible, starting with reasoning about performance. However, I'm sure I'll have missed something, or some new feature or switch will come along that I don't address. So, *caveat emptor*!

If your hair is on fire and you need to try something now, rather than reading through a philosophical treatise first, you can jump to [Part 5](/post/query-performance-improvement-part-5) of this series, which offers thoughts on indexing and rewriting queries that are most likely to have a direct impact on performance.

However, if you're looking to make more fundamental changes to working with a database, then I strongly recommend reading the blogs in order.

# Background

## Who cares?

You may well ask why you should care about query performance. Modern hardware is so powerful and RAM is so cheap that you can just scale up the server if things aren't going fast enough. Well, up to a point. I've been doing this for a long time and sooner or later everyone runs out of money or gets a bollocking from the CFO because of their AWS bill. At this point, it's really, really painful to try and speed up a database that has been treated like a pastebin - you're already under the cosh, and everything you need to do will take ages, cause outages and make everyone unhappy.

## Fundamental reasoning

Let's assume you don't want to get a bollocking from the CFO, and look at the issue of performance from the database's point of view.

Databases tend to be run on quite substantial servers. This is just a function of the need to evaluate and process huge amounts of data. Even horizontally scalable distributed SQL databases like Cockroach perform better on a few huge servers than on dozens of small servers.

This makes it harder to scale up, because for most substantial applications, you're already using servers above the median size. But it also means that you have quite a powerful server that you can leverage to get answers quickly.

## Use the database as much as necessary, but no more than that

### Get answers, not data

The first thing to do is to use the database to give answers, rather than provide data. This is contextual: if the customer is asking for a statement of transactions, you will obviously need to to return more data than if they are asking for a balance. You *could* just have one function that did both, but that would make every balance enquiry less efficient and slower than it needs to be.

### Populating lookup tables

Static lookup tables should only be lazy loaded once, the first time they are used. If the lookup table changes while the program is running you can do something like publish a message from PostgreSQL that the program can subscribe to, to empty the table and let it reload when next used.

### Do I even need this query?

Something else to consider (and it really isn't done often enough!) is this: do I actually need this query? Does it provide me with any value? An example is where you're writing database rows to a message queue and mark them as sent. It's frustratingly common that the developer will do a count of unsent rows before trying to write a batch to the queue. Why do the count? Why not just try to do the write and if there are no rows to process, just drop out? (If this sounds picky, counting a number of rows in a large table can be so slow that it blows your response time SLO!)

# SERIALIZABLE isolation

The principle behind SERIALIZABLE isolation is that the developer need not be concerned about manually locking rows, the database takes care of isolating each transaction from every other.

This sounds absolutely wonderful to developers, but it's important to realise that There Ain't No Such Thing As A Free Lunch: SERIALIZABLE isolation often leads to failed transactions (which can just be re-applied, but they do have to be re-applied.) So it's doing twice as much work (at least) for every failed transaction. It can also lead to all sorts of very painful contention issues. It's also important to realise that it's not just data modification that gets serialised - SELECTs get serialised, too!

My reading of the [academic paper](https://arxiv.org/pdf/1208.4179) behind PostgreSQL's serialisation mechanism shows that PostgreSQL will fail any transaction that could *potentially* have a conflict, not just those that actually *do* have a conflict.

Furthermore, if query A gets considered for a conflict by query B, query A will be considered for a conflict by other queries as long as a query B is still running - even if query A has actually already completed! This means a lot of queries can fail for no reason.

As an aside, I've been doing database "stuff" for 40 years now, and I'd never come across SERIALIZABLE (or its equivalent) until my last job, where I discovered why I'd never come across it before, and why I hope I never come across it again. Save yourself from a world full of pain and don't use SERIALIZABLE (or its equivalent in other databases.)

# ANALYZE

PostgreSQL has a cost-based optimiser that works out the most cost-efficient way to perform a query. The optimiser depends on a number of statistics to decide whether or not to use indexes, which indexes, join order, etc.

The mechanism to keep these statistics in shape is the [ANALYZE](https://www.postgresql.org/docs/current/sql-analyze.html) command.

For example, let's assume we have a table with an index, but the table only has 2 rows in it. The optimiser will always do a sequential scan of this table, as it is faster than traversing the index. However, if the table somehow gets 2 billion rows in it and the statistics are not updated, the optimiser will run all queries as sequential scans, even if the index would be much more efficient.

Current versions of PostgreSQL tend to run auto-analyze, but it's worth forcing a manual ANALYZE on every table on a schedule based on the volatility of the table. Static lookup tables can be analyzed once a month, large and steadily growing tables once a week, and tables that oscillate wildly or grow rapidly should be done at least daily. This is primarily a safety measure and is not a sufficient substitute for auto-analyze.

# 3NF / 4NF

One frequently observed suggestion for improving performance is to denormalise your database to avoid the cost of joining tables. In truth, PostgreSQL has a plethora of different strategies for joining tables, so an individual join is unlikely to be problematic as long as optimiser statistics are current. Relational databases have honed and polished their handling of joins over decades.

It only becomes likely to cause an issue when you have five or more joins in a query involving large numbers of rows and poor indexing, which is a combination I personally have rarely encountered. I would suggest that if you are regularly joining five or more times in a single query, you may have a different issue with your database design.

# Summary

In this post, we looked at:
- why you should optimise rather than just scaling up (or out)
- principles of efficient database use
- the benefits of removing queries
- why you should avoid using SERIALIZABLE isolation
- the basics of keeping optimiser statistics
- why normalising a database is normally (ha! ha!) not a problem

---

Next: [Bloat and Toast](/post/query-performance-improvement-part-2)
