---
title:  "An Introduction to Data Warehouse Query Strategies, Part 1 of N"
date:   2018-06-22 10:00AM
excerpt: "An introduction to query strategies for data warehouse platforms, such as Redshift, Snowflake and Teradata"
categories: [sql]
---
I'm working with a few clients on either building out a new data warehouse, or with new analysts accessing an existing data warehouse for the first time. In either case, it's good to be aware of a few common principles when querying large data warehouse tables, especially those on data warehouse specific platforms such as Redshift, Snowflake and Teradata.

### Don't join fact tables
- Unintentional product joins
    - Can create redistribution issues on MPP databases
- Instead:
    - Use Common-Table-Expressions (CTE) to pre-aggregate fact tables to a common level

```sql
with metric_a as
(
	select txn_date, sum(fct_col) as metric_a
 	from fct_table_a
 	group by 1
),
with metric_b as
(
	select txn_date, sum(fct_col) as metric_b
	from fct_table_b
	group by 1
)
select
    coalesce(ma.txn_date, mb.txn_date) as txn_date,
    coalesce(ma.metric_a,0) as metric_a,
    coalesce(mb.metric_b, 0) as metric_b
from
    metric_a ma
    full outer join
    metric_b mb on ma.txn_date = mb.txn_date

```

### Use CTEs
- They’re great to reduce complexity and to increase reuse
- Use instead of subqueries
- Use as modular building blocks

```sql
with complicated_logic as
(
	...
),
with metric_a as
(
	select ...
	from complicated_logic
	where some_parameter = 'something'
),
with metric_b as
(
	select ...
	from complicated_logic
	where some_parameter = 'something else'
)
select
...
from
    metric_a ma
    full outer join
    metric_b mb on ma.txn_date = mb.txn_date
```

### Big Table First
- Fact on the left, Dimension on the right
- Use inner join where possible

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

### Don't Right Outer Join Anythin Ever
- If not possible, use left outer join, never right outer join
    - Rearrange!

### Filter Early
- Always filter fact tables on partition key
    - Usually the fact date
    - Potentially significant speed up for partitioned tables

```sql
with metric_a as
(
	select txn_date, sum(fct_col) as metric_a
 	from fct_table_a
    where
        txn_date between '2018-01-01' and '2018-05-31'
    group by 1
),
with metric_b as
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
```

### Don't Filter on Computed Columns
- Can cause full table scans despite filter
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

Better, use your date dimension:

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
### For The Same Reason, Don't Join on Computed Columns

Don't:
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
