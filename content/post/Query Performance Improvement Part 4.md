+++
categories = ['databases', 'postgresql', 'performance']
date = '2025-04-23T12:07:01+01:00'
draft = false
tags = ['postgresql', 'performance']
title = 'Query Performance Improvement Part 4'
+++

# Overview

[Part 1](/post/query-performance-improvement-part-1) looks at some principles and introduces some of the tools in your arsenal.

[Part 2](/post/query-performance-improvement-part-2) covers the dreaded PostgreSQL bloat issue as well as how TOAST can help performance, or hinder it.

[Part 3](/post/query-performance-improvement-part-3) dives deeper into identifying, investigating and mitigating slow queries.

Part 4 (this post) covers indexing.

[Part 5](/post/query-performance-improvement-part-5) covers rewriting queries.

# TL;DR

- If you see "Filter:" or "Rows Removed" in your explain plan, you are likely to be using the wrong index
- Composite indexes can be much faster than multiple single indexes
- Partial indexes reduce the height and size of the index, making them faster
- Covering indexes can speed queries even further
- The bigger the table, the more likely it is that an index will add value
- Small tables are often sequentially scanned, whether indexed or not
- Small to medium tables tend to benefit from single column indexes, less from more complex indexes
- Large tables tend to benefit more from complex indexes
- Always test before pushing to production!

# Indexes

Indexes are one of the most effective tools to improve query performance. Indexes work best when they return a small subset of data and are small compared to row size. So an 8-byte integer index on a 2kB row is more efficient than a 1kB text index on a 1.5kB row.

However, sometimes the gains from composite indexes or covering indexes can outweigh the efficiency ratio.

## The RIGHT index?

This is often (very often!) the big knowledge gap in query tuning: an inexperienced person will examine a query plan, see that it's using an index and assume that the query is therefore optimised. This is often not the case. If you carefully examine the explain plan and find that it says "Filter: " and / or a large number of "Rows Removed", then your indexing is almost certainly suboptimal.

This very much depends on the size of the data set being queried. For small tables, PostgreSQL will often just scan the table because it is faster than traversing the index. As the table grows, the index becomes more valuable. In medium-sized tables, a single column index and discarding rows can be faster than the overhead of traversing a larger composite index. You can possibly mitigate this by having a partial index covering only the section of the table you are interested in. Only with very large tables is it likely for a complex index to have an unequivocal effect.

## Composite indexes

PostgreSQL can use multiple single-column indexes to satisfy a query.

```
bench=# \d pgbench_accounts
              Table "public.pgbench_accounts"
  Column  |     Type      | Collation | Nullable | Default
----------+---------------+-----------+----------+---------
 aid      | integer       |           | not null |
 bid      | integer       |           |          |
 abalance | integer       |           |          |
 filler   | character(84) |           |          |
Indexes:
    "pgbench_accounts_pkey" PRIMARY KEY, btree (aid)
Foreign-key constraints:
    "pgbench_accounts_bid_fkey" FOREIGN KEY (bid) REFERENCES pgbench_branches(bid)
Referenced by:
    TABLE "pgbench_history" CONSTRAINT "pgbench_history_aid_fkey" FOREIGN KEY (aid) REFERENCES pgbench_accounts(aid)

bench=# explain (analyze, buffers, costs, settings, verbose) select count(*) from pgbench_accounts where aid >0 and aid < 194999 and abalance < -4500 and abalance > -5000;
                                                                                     QUERY PLAN
-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 Finalize Aggregate  (cost=60200.96..60200.97 rows=1 width=8) (actual time=52.971..57.025 rows=1 loops=1)
   Output: count(*)
   Buffers: shared hit=14648
   ->  Gather  (cost=60200.75..60200.96 rows=2 width=8) (actual time=52.671..57.005 rows=3 loops=1)
         Output: (PARTIAL count(*))
         Workers Planned: 2
         Workers Launched: 2
         Buffers: shared hit=14648
         ->  Partial Aggregate  (cost=59200.75..59200.76 rows=1 width=8) (actual time=29.864..29.866 rows=1 loops=3)
               Output: PARTIAL count(*)
               Buffers: shared hit=14648
               Worker 0:  actual time=20.344..20.346 rows=1 loops=1
                 Buffers: shared hit=2914
               Worker 1:  actual time=20.943..20.945 rows=1 loops=1
                 Buffers: shared hit=2944
               ->  Parallel Index Scan using pgbench_accounts_pkey on public.pgbench_accounts  (cost=0.43..59197.26 rows=1394 width=0) (actual time=0.254..29.601 rows=1127 loops=3)
                     Output: aid, bid, abalance, filler
                     Index Cond: ((pgbench_accounts.aid > 0) AND (pgbench_accounts.aid < 194999))
                     Filter: ((pgbench_accounts.abalance < '-4500'::integer) AND (pgbench_accounts.abalance > '-5000'::integer))
                     Rows Removed by Filter: 63872
                     Buffers: shared hit=14648
                     Worker 0:  actual time=0.307..20.183 rows=674 loops=1
                       Buffers: shared hit=2914
                     Worker 1:  actual time=0.369..20.770 rows=678 loops=1
                       Buffers: shared hit=2944
 Planning:
   Buffers: shared hit=10
 Planning Time: 0.846 ms
 Execution Time: 57.097 ms
(29 rows)

Time: 58.965 ms

bench=# \d pgbench_accounts
              Table "public.pgbench_accounts"
  Column  |     Type      | Collation | Nullable | Default
----------+---------------+-----------+----------+---------
 aid      | integer       |           | not null |
 bid      | integer       |           |          |
 abalance | integer       |           |          |
 filler   | character(84) |           |          |
Indexes:
    "pgbench_accounts_pkey" PRIMARY KEY, btree (aid)
    "i_pa2" btree (abalance)
Foreign-key constraints:
    "pgbench_accounts_bid_fkey" FOREIGN KEY (bid) REFERENCES pgbench_branches(bid)
Referenced by:
    TABLE "pgbench_history" CONSTRAINT "pgbench_history_aid_fkey" FOREIGN KEY (aid) REFERENCES pgbench_accounts(aid)

bench=# explain (analyze, buffers, costs, settings, verbose) select count(*) from pgbench_accounts where aid >0 and aid < 194999 and abalance < -4500 and abalance > -5000;
                                                                                          QUERY PLAN
----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 Aggregate  (cost=18411.18..18411.19 rows=1 width=8) (actual time=99.326..99.328 rows=1 loops=1)
   Output: count(*)
   Buffers: shared hit=4763 read=161
   ->  Bitmap Heap Scan on public.pgbench_accounts  (cost=6475.99..18402.82 rows=3345 width=0) (actual time=77.222..98.977 rows=3382 loops=1)
         Recheck Cond: ((pgbench_accounts.abalance < '-4500'::integer) AND (pgbench_accounts.abalance > '-5000'::integer) AND (pgbench_accounts.aid > 0) AND (pgbench_accounts.aid < 194999))
         Rows Removed by Index Recheck: 192252
         Heap Blocks: exact=938 lossy=3288
         Buffers: shared hit=4763 read=161
         ->  BitmapAnd  (cost=6475.99..6475.99 rows=3345 width=0) (actual time=76.538..76.539 rows=0 loops=1)
               Buffers: shared hit=535 read=161
               ->  Bitmap Index Scan on i_pa2  (cost=0.00..2222.09 rows=164966 width=0) (actual time=52.544..52.544 rows=178341 loops=1)
                     Index Cond: ((pgbench_accounts.abalance < '-4500'::integer) AND (pgbench_accounts.abalance > '-5000'::integer))
                     Buffers: shared read=161
               ->  Bitmap Index Scan on pgbench_accounts_pkey  (cost=0.00..4251.98 rows=202754 width=0) (actual time=16.240..16.240 rows=194998 loops=1)
                     Index Cond: ((pgbench_accounts.aid > 0) AND (pgbench_accounts.aid < 194999))
                     Buffers: shared hit=535
 Planning:
   Buffers: shared hit=16 read=1
 Planning Time: 0.904 ms
 Execution Time: 99.410 ms
(20 rows)

Time: 104.037 ms
```

In the first query above, we find all the matching `aid` values and then drop rows where the `abalance` is positive. In the second query, two indexes are bitmap scanned to identify candidate rows and then the Index Recheck validates the data for each row. Because this is not a very large table, the second approach is actually slower as it has to eliminate more data.

Now let's consider what happens when you have a composite key.

```
bench=# \d pgbench_accounts
              Table "public.pgbench_accounts"
  Column  |     Type      | Collation | Nullable | Default
----------+---------------+-----------+----------+---------
 aid      | integer       |           | not null |
 bid      | integer       |           |          |
 abalance | integer       |           |          |
 filler   | character(84) |           |          |
Indexes:
    "pgbench_accounts_pkey" PRIMARY KEY, btree (aid)
    "i_pa2" btree (aid, abalance)
Foreign-key constraints:
    "pgbench_accounts_bid_fkey" FOREIGN KEY (bid) REFERENCES pgbench_branches(bid)
Referenced by:
    TABLE "pgbench_history" CONSTRAINT "pgbench_history_aid_fkey" FOREIGN KEY (aid) REFERENCES pgbench_accounts(aid)

bench=# explain (analyze, buffers, costs, settings, verbose) select count(*) from pgbench_accounts where aid >0 and aid < 194999 and abalance < -4500 and abalance > -5000;
                                                                                         QUERY PLAN
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 Aggregate  (cost=11879.83..11879.84 rows=1 width=8) (actual time=27.790..27.791 rows=1 loops=1)
   Output: count(*)
   Buffers: shared hit=2743 read=532
   ->  Index Only Scan using i_pa2 on public.pgbench_accounts  (cost=0.43..11871.47 rows=3345 width=0) (actual time=0.126..27.198 rows=3382 loops=1)
         Output: aid, abalance
         Index Cond: ((pgbench_accounts.aid > 0) AND (pgbench_accounts.aid < 194999) AND (pgbench_accounts.abalance < '-4500'::integer) AND (pgbench_accounts.abalance > '-5000'::integer))
         Heap Fetches: 3382
         Buffers: shared hit=2743 read=532
 Planning:
   Buffers: shared hit=12 read=4
 Planning Time: 1.204 ms
 Execution Time: 27.839 ms
(12 rows)

Time: 30.328 ms
```

As you can see, it is substantially quicker.

## Partial indexes

There is one other optimisation we can consider: if you are only interested in a subset of rows, then only index those rows.

```
bench=# create index i_pa2 on pgbench_accounts(aid, abalance) where aid > 0 and aid < 194999 and abalance < -4500 and abalance > -5000;
CREATE INDEX
Time: 4182.695 ms (00:04.183)
bench=# \d pgbench_accounts
              Table "public.pgbench_accounts"
  Column  |     Type      | Collation | Nullable | Default
----------+---------------+-----------+----------+---------
 aid      | integer       |           | not null |
 bid      | integer       |           |          |
 abalance | integer       |           |          |
 filler   | character(84) |           |          |
Indexes:
    "pgbench_accounts_pkey" PRIMARY KEY, btree (aid)
    "i_pa2" btree (aid, abalance) WHERE aid > 0 AND aid < 194999 AND abalance < '-4500'::integer AND abalance > '-5000'::integer
Foreign-key constraints:
    "pgbench_accounts_bid_fkey" FOREIGN KEY (bid) REFERENCES pgbench_branches(bid)
Referenced by:
    TABLE "pgbench_history" CONSTRAINT "pgbench_history_aid_fkey" FOREIGN KEY (aid) REFERENCES pgbench_accounts(aid)

bench=# explain (analyze, buffers, costs, settings, verbose) select count(*) from pgbench_accounts where aid >0 and aid < 194999 and abalance < -4500 and abalance > -5000;
                                                                    QUERY PLAN
---------------------------------------------------------------------------------------------------------------------------------------------------
 Aggregate  (cost=6679.09..6679.10 rows=1 width=8) (actual time=8.584..8.586 rows=1 loops=1)
   Output: count(*)
   Buffers: shared hit=2740 read=11
   ->  Index Only Scan using i_pa2 on public.pgbench_accounts  (cost=0.28..6670.73 rows=3345 width=0) (actual time=0.458..7.853 rows=3382 loops=1)
         Output: aid, abalance
         Heap Fetches: 3382
         Buffers: shared hit=2740 read=11
 Planning:
   Buffers: shared hit=36 read=1
 Planning Time: 1.261 ms
 Execution Time: 8.659 ms
(11 rows)

Time: 13.911 ms
```
Because the query has less index to traverse, the query is considerably quicker.

Obviously, this cannot be applied to all indexes, but the gains can be very useful if it is usable.

## Covering indexes

Another advanced optimisation is a covering index. This means that the index covers all the columns that are returned by the query, so that the query does not have to examine data rows.
```
bench=# \d pgbench_accounts
              Table "public.pgbench_accounts"
  Column  |     Type      | Collation | Nullable | Default
----------+---------------+-----------+----------+---------
 aid      | integer       |           | not null |
 bid      | integer       |           |          |
 abalance | integer       |           |          |
 filler   | character(84) |           |          |
Indexes:
    "pgbench_accounts_pkey" PRIMARY KEY, btree (aid)
    "i_pa2" btree (aid, abalance) WHERE aid > 0 AND aid < 194999 AND abalance < '-4500'::integer AND abalance > '-5000'::integer
Foreign-key constraints:
    "pgbench_accounts_bid_fkey" FOREIGN KEY (bid) REFERENCES pgbench_branches(bid)
Referenced by:
    TABLE "pgbench_history" CONSTRAINT "pgbench_history_aid_fkey" FOREIGN KEY (aid) REFERENCES pgbench_accounts(aid)

bench=# explain (analyze, buffers, costs, settings, verbose) select count(distinct bid) from pgbench_accounts where aid >0 and aid < 194999 and abalance < -4500 and abalance > -5000;
                                                                     QUERY PLAN
-----------------------------------------------------------------------------------------------------------------------------------------------------
 Aggregate  (cost=6883.26..6883.27 rows=1 width=8) (actual time=17.342..17.344 rows=1 loops=1)
   Output: count(DISTINCT bid)
   Buffers: shared hit=2324
   ->  Sort  (cost=6866.54..6874.90 rows=3345 width=4) (actual time=15.559..16.273 rows=3382 loops=1)
         Output: bid
         Sort Key: pgbench_accounts.bid
         Sort Method: quicksort  Memory: 97kB
         Buffers: shared hit=2324
         ->  Index Scan using i_pa2 on public.pgbench_accounts  (cost=0.28..6670.73 rows=3345 width=4) (actual time=0.036..14.742 rows=3382 loops=1)
               Output: bid
               Buffers: shared hit=2324
 Planning:
   Buffers: shared hit=27 read=1
 Planning Time: 1.119 ms
 Execution Time: 17.460 ms
(15 rows)

Time: 25.948 ms
bench=# drop index i_pa2;
DROP INDEX
Time: 5.494 ms
bench=# create index i_pa2 on pgbench_accounts(aid, abalance, bid) where aid > 0 and aid < 194999 and abalance < -4500 and abalance > -5000;
CREATE INDEX
Time: 3619.871 ms (00:03.620)
bench=# explain (analyze, buffers, costs, settings, verbose) select count(distinct bid) from pgbench_accounts where aid >0 and aid < 194999 and abalance < -4500 and abalance > -5000;
                                                                       QUERY PLAN
---------------------------------------------------------------------------------------------------------------------------------------------------------
 Aggregate  (cost=6895.26..6895.27 rows=1 width=8) (actual time=7.440..7.443 rows=1 loops=1)
   Output: count(DISTINCT bid)
   Buffers: shared hit=2740 read=14
   ->  Sort  (cost=6878.54..6886.90 rows=3345 width=4) (actual time=6.157..6.699 rows=3382 loops=1)
         Output: bid
         Sort Key: pgbench_accounts.bid
         Sort Method: quicksort  Memory: 97kB
         Buffers: shared hit=2740 read=14
         ->  Index Only Scan using i_pa2 on public.pgbench_accounts  (cost=0.28..6682.73 rows=3345 width=4) (actual time=0.135..5.331 rows=3382 loops=1)
               Output: bid
               Heap Fetches: 3382
               Buffers: shared hit=2740 read=14
 Planning:
   Buffers: shared hit=48 read=1
 Planning Time: 2.134 ms
 Execution Time: 7.512 ms
(16 rows)

Time: 11.057 ms
```
In the first query, we have our super-efficient index, but once the rows are identified, the data still has to be read to obtain the values for the `bid` column.

If we create a covering index, the query speeds up because reading the data is not required - the significant change in the query plan is from `Index Scan using i_pa2` to `Index Only Scan using i_pa2`.

In general, you should only use covering indexes on values that can fit into 20 bytes[^1] or less. Beyond this, the extra index size contends more strongly with the benefit.

Also note that there are two forms for the creation of a covering index:
- CREATE INDEX tab_x_y ON tab(x,y);
- CREATE INDEX tab_x_y ON tab(x) INCLUDE (y);

The first form should be used when y is a data type understood by the index. You should use this because PostgreSQL compresses duplicate values in the index, but does not as yet do this for the second form.

The second form should only be used for a data type that is not understood by the index, because the index stores it "as is" and does not evaluate it for compression.

[^1]: 20 is an arbitrary number chosen totally at random, but there's no point in storing War And Peace or the Encyclopedia Britannica in your index. Small is beautiful.

# Summary

In this post we looked at:
- Warning signs that indicate possible suboptimal indexes
- Composite versus multiple single indexes
- Partial indexes
- Covering indexes

---

Previous: [Identifying, investigting and mitigating](/post/query-performance-improvement-part-3) | Next: [Rewriting queries](/post/query-performance-improvement-part-5/)
