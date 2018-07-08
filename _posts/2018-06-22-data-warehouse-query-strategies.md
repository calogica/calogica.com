---
title:  "An Opinionated Introduction to Data Warehouse Query Strategies, Part 1 of N"
date:   2018-06-22 10:00AM
excerpt: "An opinionated introduction to query strategies for data warehouse platforms, such as Redshift, Snowflake and Teradata"
categories: [sql]
comments: true
---
I often work with new clients on building out a new data warehouse, or with new analysts accessing an existing data warehouse for the first time. In either case, it's good to be aware of a few common principles when querying large data warehouse tables, especially those on data warehouse specific platforms such as Redshift, Snowflake and Teradata.

In this and likely a few more related posts, we'll explore some common query strategies and pitfalls when dealing with data warehouse databases.

## Don't join fact tables
Joining large fact tables (such as an `Orders` and a `Shipments` fact table in an eCommerce data warehouse) together in attempt to create a report combining a few metrics potentially leads to issues that are later hard to track down.

#### Don't :(
```sql
select
    o.order_date,
    sum(o.order_cnt) as order_cnt,
    sum(s.shipment_cnt) as shipment_cnt
from
    fct_orders o
    join
    fct_shipments s on o.order_id = s.order_id
group by 1
```

### What do you mean? What's wrong with this query?
- Well, for one we might expose ourselves to the possibility of **unintentional product joins** - if not now, then down the road.
For example, you might assume (or even know) that the `Shipments` fact table is at the *Order* level, i.e. each row in the `Shipments` fact is for exactly one `Order`. However, unbeknownst to you, the distribution center modifies the `Shipments` process such that we find ourselves with *multiple* shipments per order for a non-trivial number of orders.
In that case, the right side of the join to `Shipments` now not only likely creates query plan complications for our database (see below), but it also causes our query to return **duplicate** rows for each order from the left side of the join.
While in some cases, this may be intentional (in which case the query should probably be explicitly structured that way), in most cases this is a subtle and hard-to-track-down bug.

- Secondly, on [MPP databases](https://databases.looker.com/analytical) where table distribution is user managed (e.g. Redshift), unless both fact tables are distributed on the same key and joined on the common distribution key, this likely causes redistribution issues as one of the large fact table now needs to be redistributed on a common key. Depending on the size of the result set, this may or may not be possible on your system.

So, what to do instead?
Use **Common-Table-Expressions** ([CTE](https://docs.aws.amazon.com/redshift/latest/dg/r_WITH_clause.html)) to pre-aggregate fact tables to a common level:

#### Do :)
```sql
with orders as
(
    select
        order_date,
        sum(order_cnt) as order_cnt
    from
        fct_orders
    group by 1
),
shipments as
(
    select     
        order_date,
        sum(shipment_cnt) as shipment_cnt
    from
        fct_shipments
    group by 1
)
select
    o.order_date as order_date,
    o.order_cnt as order_cnt,
    coalesce(s.shipment_cnt, 0) as shipment_cnt
from
    orders o
    -- we use a "left outer join" since not all orders
    -- for a given order date might have shipped already
    left outer join
    shipments s on o.order_date = s.order_date

```

In short, redistribution is costly, and sometimes can be avoided by restructuring queries.

And in general,
## Use CTEs
Generally, avoid subqueries and use CTEs instead. CTEs are a great way to reduce complexity and to enable reuse of code blocks within your query. Using CTEs, you can easily structure queries using modular building blocks and treat your SQL like a software developer, while still maintaining set logic.

#### Do :)
```sql
with complicated_logic as
(
	...
),
metric_a as
(
    select ...
    from complicated_logic
    where some_parameter = 'something'
),
metric_b as
(
    select ...
    from complicated_logic
    where some_parameter = 'something else'
)
select
    coalesce(ma.txn_date, mb.txn_date) as txn_date,
    coalesce(ma.metric_a,0) as metric_a,
    coalesce(mb.metric_b, 0) as metric_b
from
    metric_a ma
    -- here, we use a "full outer join"
    -- since metric a and metric b might not have
    -- values for every transaction date
    full outer join
    metric_b mb on ma.txn_date = mb.txn_date
```
On some platforms, CTEs can also be used recursively. But to understand recursion, you first must understand [recursion](https://www.google.com/search?q=recursion).

## Big Table First
Simply put, in a data warehouse query the fact table goes on the `left` side of the join, any dimensions go on the `right` side.
This makes the intent of the query more explicit, since we're clearly saying we're interested in the sum of a fact, by one or more dimensions. The fact is always more important than the dimension, so putting it first in the `from` clause is good practice and leads to more readable and self-documenting code.

#### Do :)
```sql
select
    f.txn_date,
    u.user_type_name,
    p.product_category_name,
    sum(f.sales_amt) as sales_amt
from
    fct_sales f
    join
    dim_user u on f.user_id = u.user_id
    join
    dim_product p on f.product_id = p.product_id
group by 1,2,3
```

## Don't Right Outer Join Anything Ever
If an `inner join` is not possible due to missing data in the dimension table, use a `left outer join` to join from a fact table to a dimension table, never a `right outer join`.
The simple fix is to rearrange your query. In databases, 3 right turns don't make a left turn.

## Filter Early, Filter Often
In most cases, we're interested in a subset of facts, typically for a certain time period or date range. Thus, we want to filter our, typically, very large fact table as efficiently as possible to just work with that slice of data downstream.
For partitioned tables - and really all your fact tables should be partitioned - filtering on the partition key (`sortkey` in [Redshift lingo](https://docs.aws.amazon.com/redshift/latest/dg/t_Sorting_data.html)) can have big performance benefits.
To get the most gain out of this, we need to filter as early as possible, as close to the fact table as possible.

#### Do :)
```sql
with metric_a as
(
    select
        txn_date,
        sum(fct_col) as metric_a
	from
        fct_table_a
    where
        txn_date between '2018-01-01' and '2018-05-31'
    group by 1
),
metric_b as
(
    ...
)
select
    coalesce(ma.txn_date, mb.txn_date) as txn_date,
    coalesce(ma.metric_a,0) as metric_a,
    coalesce(mb.metric_b, 0) as metric_b
from
    metric_a ma
    full outer join
    metric_b mb on ma.txn_date = mb.txn_date
-- don't filter here since that might not tell the query optimizer
-- to take advantage of partitioning
```

## Don't Filter on Computed Columns
Where possible, avoid filtering on computed columns. This can throw off the query optimizer on some platforms and can cause your nicely early-filtered query to perform a full table scans despite the filter.

So, instead of this:

#### Don't :(
```sql
select
    txn_date,
    sum(fct_col) as metric_a
from
    fct_table_a
where
    date_trunc(‘month’, txn_date) between '2018-01-01' and '2018-05-01'
group by 1
```

You're often better off using your [date dimension](http://radacad.com/do-you-need-a-date-dimension):

#### Do :)
```sql
select
    txn_date,
    sum(fct_col) as metric_a
from
    fct_table_a f
    join
    dim_date on f.txn_date = d.calendar_date
where
    d.month_start_date between '2018-01-01' and '2018-05-01'
group by 1
```


For The Same Reason,
## Don't Join on Computed Columns

I see, and do, this all the time, but generally this is a smell that we need to preprocess our data earlier in the pipeline.

#### Don't :(
```sql
select
    txn_date,
    sum(fct_col) as metric_a
from
    fct_table_a f
    join
    dim_user u on lower(f.email_address) = lower(u.email_address)
group by 1
```

Do you have any more opinionated data warehouse query tips? Let me know!

I know I do...to be continued.
