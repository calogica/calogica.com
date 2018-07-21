---
title:  "Random Sampling Within Groups using SQL"
date:   2018-07-21 8:00AM
excerpt: "How to randomly select rows within a subset or group of your data using SQL"
categories: [sql]
comments: true
---
Here's just a quick SQL tip I came across today while working on a sample dataset for a take-home exercise.

The challenge was: how do I randomly select some `N` number of rows from a large dataset within a group. In my case, I want a random sample of 1,000 customers by sign up year.

Turns out, you can do that using our friend the **analytical function** aka _window function_ in SQL. I've written more about these handy functions [here](/sql/2018/07/01/sql-functions-for-data-analyst-interviews.html) and [here](/sql/2018/06/22/data-warehouse-query-strategies.html).

Assuming I have a dataset consisting of `user_id` and `signup_date` with signup dates ranging from `2017-01-01` to `2018-07-20`:

```
| user_id                          | signup_date |
|----------------------------------|-------------|
| ed4f25103c7f6d9c41e14c047d2ff930 | 2017-02-04  |
| 7751a2c62c91383aec0e9be07457b5b0 | 2017-03-19  |
| eb63c09470b6d1564fe77132499f921e | 2017-07-11  |
| 84239f8c064ae3dfd7e61c139dbde6d5 | 2017-04-25  |
| 3a7b7ce201ebbe36d6599c0b9f859bad | 2017-04-18  |
| abfb263db894248709e224fb9a683a61 | 2017-09-24  |
| b74f62527bce6c1bfaf93d6104506b05 | 2017-04-09  |
| 3bf1bb77cc78d921ace25cb16cc37a54 | 2017-07-27  |
| 4b72f06bbef45f45494c4214d2cebd9c | 2017-04-10  |
| 6d0c208da45034fe6a0a2a2db04f117a | 2017-08-21  |
...
```

Let's say we want to pick out `5` random rows by signup _year_ (or _month_, _week_, whatever), we will need to:
- Create a random row number for each `user_id` that resets for each of my _periods_ or _groups_.
We do that by ordering the `row_number()` function using the `random()` [function](https://docs.snowflake.net/manuals/sql-reference/functions/random.html).
Note: `random()` can be parameterized with a `seed` but it looks to me that because of the parallel nature of data warehouse platforms like Snowflake and Redshift, this only leads to repeatable results if your code is only executed on one node (like the leader node).  
- Select `N` of those rows filtering on our new random row number.

For example:
```sql
with randomly_sorted_users as
(
    select
        user_id,
        signup_date,
        row_number() over(partition by date_trunc('year', signup_date)
                            order by random()) as random_sort
    from
        user_table
)
select
    user_id,
    signup_date
from
    randomly_sorted_users
where
    random_sort <= 5
```
This will yield 10 rows in total, 5 chosen randomly by _signup year_:
```
| #  | user_id                          | signup_date |
|----|----------------------------------|-------------|
| 1  | e58975cdb6f4b052389b199f623319dd | 2017-10-15  |
| 2  | c7253ee384f5be436fef0b900c9e5125 | 2017-05-03  |
| 3  | 1400ffe327b1b15130fe85535eee58ba | 2017-03-05  |
| 4  | 1079a421f9a59e1716082424310f6dc0 | 2017-12-15  |
| 5  | f4fed873df3786d04cdef1ccce59eb36 | 2017-02-21  |
| 6  | db3a60a5d55fc8a4aa20f5d8a2f276af | 2018-01-25  |
| 7  | 9e991194118d3e4db471106358825685 | 2018-04-11  |
| 8  | 008a3b0f05c601ea39f0e893d4e11043 | 2018-02-09  |
| 9  | 7e40c4cce72ae34ca0e6e4e39efdb514 | 2018-03-04  |
| 10 | 1b9d2c64344629aebf2349d328e76aae | 2018-06-30  |
```
