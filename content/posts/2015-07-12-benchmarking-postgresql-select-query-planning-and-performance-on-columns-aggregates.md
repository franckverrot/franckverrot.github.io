---
title: "Benchmarking PostgreSQL's SELECT Query Planning and Performance on Columns Aggregates"
date: 2015-07-12 22:50:00
categories:
  - postgresql
  - sql
  - performance
slug: "benchmarking-postgresql-select-query-planning-and-performance-on-columns-aggregates"
tags:
  - postgresql
  - performance
---

Executing *a single query* doesn’t always literally mean *executing exactly ONE query*. Sometimes PostgreSQL (like any other relation database) will have to extract parts of your query (subqueries) and re-execute it as many times as needed to generate the results you are expecting it to generate. This article shows how `N+1` issues can also happen without an explicit intent to produce them.

<!--more-->

# Introduction to the problem


Computing aggregated columns is a common task, sometimes requiring the relational engine to go through more than one table: counting comments on blog posts, counting favorites on tweets, …

This task can be done in many ways, here are the first three that come to my mind:

1. **Using multiple queries**

```sql
# Query all `Posts`
SELECT * from posts

# For each `Post`, query its `Comments` count
SELECT count(1) from comments where post_id = ?
```

2. **Using a `JOIN`**

```sql
SELECT
  posts.id,
  count(1)
FROM posts
  LEFT OUTER JOIN comments
    ON posts.id = comments.post_id
GROUP BY posts.id
```


3. **Substitute a column for a subquery**

```sql
SELECT
  posts.id,
  (SELECT count(1)
    FROM comments
    WHERE comments.post_id = post.id) comments_counts
FROM posts
```

While many developers now about the performance implications and issues of solution #1 (the `N+1` queries it will generate — where `N` is the number of blog posts), it isn’t obvious what would be the difference between solution 2 and 3, especially when it comes to dealing with large datasets.

# Running the benchmarks

Let’s take those two implementations (the one based on `JOIN`s and the one using a subquery), and analyze the difference<sup>[1](#github)</sup>.

For that purpose, let’s use `EXPLAIN ANALYZE` on both of them. We will use the `LIMIT` clause and iteratively restrict the dataset to see the complexity trend: are we going to observe a logarithmic growth? linear growth? constant time?

## Using `JOIN`s
Let’s first analyze the `JOIN` strategy:


```sql
 explain analyze select p.id, count(c.post_id) from posts p left outer join comments c on p.id = c.post_id group by 1; — no limit yet
UERY PLAN
—————————
HashAggregate  (cost=1768.05..1773.05 rows=500 width=4) (actual time=28.696..28.749 rows=500 loops=1)
  Group Key: p.id
  ->  Hash Right Join  (cost=14.25..1501.65 rows=53280 width=4) (actual time=0.207..18.019 rows=49901 loops=1)
        Hash Cond: (c.post_id = p.id)
        ->  Seq Scan on comments c  (cost=0.00..754.80 rows=53280 width=4) (actual time=0.007..4.832 rows=50000 loops=1)
        ->  Hash  (cost=8.00..8.00 rows=500 width=4) (actual time=0.192..0.192 rows=500 loops=1)
              Buckets: 1024  Batches: 1  Memory Usage: 18kB
              ->  Seq Scan on posts p  (cost=0.00..8.00 rows=500 width=4) (actual time=0.012..0.097 rows=500 loops=1)
Planning time: 0.121 ms
Execution time: 28.825 ms
```

The plan for that query is pretty straightforward:

1. The planner first executes a sequential scan (i.e.: reads all the records out of the `posts` table, which is the smallest table) and stores rows with the same `id` in the same bucket (each post has a unique id though, so this doesn't seem really meaningful in our case, but this is a performance trick to avoid hashing the largest table and overusing the memory). Accessing posts later on, in memory, will be achieved in constant time<sup>[2](#constant-time)</sup>
2. Then it will scan all the `comments`, and iteratively merge the two datasets using the pre-computed hash key (`post_id`) and the [`Hash Right Join` function][hrj]. At this point, a dataset of `N * M` records (`N` for the number of posts, `M` for the number of comments) could exist. However, the hash algorithm we’re using don’t require to have the whole dataset in memory and only buffers the aggregated results
3. Finally, we aggregate this dataset using the `HashAggregate` function.

Using `JOIN` makes use of `HashAggregate`, which is very efficient (and if both tables comes already sorted by the `id`/`post_id` columns, the planner will decide to use the `GroupAggregate` function and the query will be even more efficient).

This solution is pretty straightforward but having to use (and write) a combination of a `LEFT OUTER JOIN` and a `GROUP BY` clause can sometimes be scary for those unwilling to deal with any kind of SQL statements.

ORMs don’t particularly make this easy on developers. As an example, let’s take a look at `Ruby on Rails`’s `Active Record`’s DSL (*Domain Specific Language*):

```ruby
Post.
  joins(“left outer join comments on posts.id = comments.post_id”).
  select(“posts.id, count(1) comments_count”).
  group_by(“1”)
```

Some people will probably want to switch to using raw SQL (with `ActiveRecord::Base.connection#execute`) but this limitation is inherent to using an ORM.

## Using a subquery

Now onto the subquery. The solution looks seducing at first sight: a query nested within another: no grouping, no joining, just a simple `WHERE` clause from the parent plan (not sure about the wording here).

```sql
# explain analyze select p.id, (select count(1) from comments c where c.post_id = p.id) from posts p;
 Seq Scan on posts p  (cost=0.00..444345.50 rows=500 width=4) (actual time=5.110..1906.266 rows=500 loops=1)
   SubPlan 1
     ->  Aggregate  (cost=888.66..888.67 rows=1 width=0) (actual time=3.810..3.810 rows=1 loops=500)
           ->  Seq Scan on comments c  (cost=0.00..888.00 rows=266 width=0) (actual time=0.029..3.793 rows=100 loops=500)
                 Filter: (post_id = p.id)
                 Rows Removed by Filter: 49900
 Planning time: 0.100 ms
 Execution time: 1906.397 ms
```

Before even trying to decipher the plan, let’s compare our `Execution time`: **1,906 ms** vs **28 ms**. It’s a **6,600%** increase…

The plan is now very different:

1. A sequential scan is done on the `posts` table. For each of these rows, a subplan `SubPlan 1` (i.e.: another query) is going to be executed.
2. `SubPlan 1` is defined as:

  2.1. It is going to be executed a certain number of time: this is the `loops` value. We can read `71`, which matches the exact number of posts from our `posts` table: this plan will be executed 71 times
  
  2.2. Given a post’s id, a sequential scan will be executed on the `comments` table
 
  2.3. Once all the comments matching the post’s `id` have been retrieved, the result set is aggregated using the `Aggregate` function
3. `SubPlan 1` is executed and inserted in the row’s columns.

What the planner unveiled was a `N+1` type of issue: given `N` records in a `posts` table, we will have as many subplans (or subqueries) as post records.

Using `ActiveRecord` will require developers to write some SQL, here are two ways of achieving the same result:

**With nested SQL**

```ruby
Post.
  select(“id,
          (select count(1)
           from comments c
           where c.post_id = posts.id) comments_count”)
```

**With AR’s underlying `Arel` “relation engine”**

```ruby
subquery = Negotiation.
        select(
          Arel::Nodes::NamedFunction.new(“count”,
          [Negotiation.arel_table[:id]])).
        where(
          Negotiation.
            arel_table[:budget_id].
            eq(Budget.arel_table[:id])).
        to_sql # not calling #to_sql will execute the query 
Budget.
  select(“id, (#{subquery})”)
```

Not really readable, the implicit (and hidden to the unwary) `N+1` issue of that strategy makes its inefficient and potentially harmful.

# Some figures

Let’s now test the performance of those queries on the same dataset. We’ll gradually increment the number of records to consider to observe the execution time growth.

![Plot and linear regression of the dataset](https://raw.githubusercontent.com/franckverrot/franckverrot.github.io/master/images/postgresql-performance-subquery-vs-join.png "Plot and linear regression of the dataset")

Applying a simple linear regression on the datasets, here are the two formulas that defines our two strategies’ performances:

* subquery(time) = 3.74724 &times; number of records + −3.51694
* join(time) = 0.00311 &times; number of records + 24.8416

<table>
<thead>
<tr>
<th>Number of records</th>
<th>Subquery<br>Execution time (Estimated)</th>
<th>`JOIN`<br>Execution time (Estimated)</th>
<th>Observation</th>
<th>JOIN faster than Subquery?</th>
</tr>
</thead>
<tbody>
<tr>
<td>1</td>
<td>± 0.23 ms</td>
<td>± 24.84 ms</td>
<td>Extrapolated &amp; Observed on the graph</td>
<td>No — 100 times slower</td>
</tr>
<tr>
<td>200</td>
<td>± 746 ms</td>
<td>± 25 ms</td>
<td>Extrapolated &amp; Observed on the graph</td>
<td>Yes — 29 times faster</td>
</tr>
<tr>
<td>1000</td>
<td>± 3,7 seconds</td>
<td>28 ms</td>
<td>Extrapolated</td>
<td>Yes — 133 times faster</td>
</tr>
<tr>
<td>10,000</td>
<td>± 37 seconds</td>
<td>± 56 ms</td>
<td>Extrapolated</td>
<td>Yes — 669 times faster</td>
</tr>
<tr>
<td>100,000</td>
<td>± 1 hour</td>
<td>± 336 ms</td>
<td>Extrapolated</td>
<td>Yes — 1115 times faster</td>
</tr>
</tbody>
</table>

# Conclusion

Most developers working with ORMs are aware of the limitations of their tools and how to avoid `N+1` in their code (using eager-loading strategies for example).
But `N+1` doesn’t only mean “sending N+1 statements to the database”, sometimes this single statement can turn into an unbelievably time-consuming statement with thousands of subplans (subqueries).

`Active Record` has an `#explain` method that can help developers understanding the generate SQL.

As an exercise to the reader, I'd suggest comparing query plans when the number of posts and the number of comments differ. For example, here's the query plan for 50k posts, and only 5k comments:

```sql
GroupAggregate  (cost=380.61..2636.48 rows=50000 width=32) (actual time=1.985..36.122 rows=50000 loops=1)
   Group Key: p.id
   ->  Merge Left Join  (cost=380.61..1886.48 rows=50000 width=32) (actual time=1.964..20.567 rows=54491 loops=1)
         Merge Cond: (p.id = c.post_id)
         ->  Index Only Scan using unique_post on posts p  (cost=0.29..1306.29 rows=50000 width=4) (actual time=0.013..10.559 rows=50000 loops=1)
               Heap Fetches: 0
         ->  Sort  (cost=380.19..392.69 rows=5000 width=32) (actual time=1.933..2.283 rows=5000 loops=1)
               Sort Key: c.post_id
               Sort Method: quicksort  Memory: 583kB
               ->  Seq Scan on comments c  (cost=0.00..73.00 rows=5000 width=32) (actual time=0.010..0.763 rows=5000 loops=1)
 Planning time: 0.810 ms
 Execution time: 38.813 ms
```

As you can see, not only it will use the unique index on the `posts` table, but also use a `GroupAggregate` given the comments are being sorted in memory.
Also, the planner can decide to build totally different plans based on the statistics it's computed on the database, so please analyze frequently.


<a name="github">1</a>: *[A GitHub’s repository is available][1] with the source code so that the results are reproducible. Please feel free to contact me if some of the numbers don’t feel right, I will keep the repository updated**.

<a name="constant-time">2</a>: Complexity is *O(log<sub>1024</sub>(N*)) (`N` being the number of `posts` records) as the planner created 1024 buckets. With a table containing billions of billons (10<sup>18</sup>) of records, the complexity is *O(6)*, simplified as *O(1)*.

[1]: https://github.com/franckverrot/postgresql-performance-subquery-vs-join
[hrj]: http://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=f4e4b3274317d9ce30de7e7e5b04dece7c4e1791
