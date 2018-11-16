---
title:  "Creating a 4-5-4 Retail Calendar using SQL and dbt"
date:   2018-11-15 7:00PM
excerpt: "Using SQL and dbt, we create a date dimension table using a retail/merchandising calendar known as a 4-5-4 calendar."
categories: [sql, dbt]
comments: true
---

Working with clients in retail eCommerce, we're often asked to analyze transactions along a type of fiscal calendar optimized for retail and merchandising. This is commonly known as the "4-5-4 Calendar", because it groups weeks (Sun-Sat) into periods of 4, 5 and 4 weeks lengths.

From the **National Retail Federation** [https://nrf.com/resources/4-5-4-calendar](https://nrf.com/resources/4-5-4-calendar):
> The 4-5-4 calendar is a guide for retailers that ensures sales comparability between years by dividing the year into months based on a 4 weeks – 5 weeks – 4 weeks format. The layout of the calendar lines up holidays and ensures the same number of Saturdays and Sundays in comparable months. Hence, like days are compared to like days for sales reporting purposes. The 4-5-4 Calendar also establishes Sales Release dates, which have historically been on the first Thursday following the month’s end. In recent years, however, as the flow of information has improved, more companies are releasing sales data earlier in the week.

For example, the retail year 2019 will start on February 3, 2019 and group the first few weeks of the year like so:


!["4-5-4 Calendar"](/assets/plots/cal454.png "4-5-4 Calendar")

As good data warehouse practitioners, we of course already have a date dimension that contains past and future dates relevant to our business, along with descriptive attributes (Day of Week etc) and groupings (Week, Month, Quarter, Year) that help us create actionable analysis. 

As an extension, it's quite common to add attributes such as holidays, as well non-calendar based attributes such as fiscal calendar based groupings.

Let's explore how to create a 4-5-4 calendar using SQL for our data warehouse (we're using Snowflake here, but this should be adaptable to Redshift or other platforms), and how to use **dbt** (our favorite data transformation tool) to make that easier.

Side note: if you're not already using **[dbt](https://www.getdbt.com/){:target="_blank"}.** to manage your data transformations, we highly recommend you take a look at it for your Redshift, Snowflake or Big Query data warehouse projects. It's been an indispensable tool for us over the last 18 months! 

### Date Dimension
First off, we need to create a date dimension for our project. We will use the helpful `date_spine` macro from the `dbt_utils` package to create a sequence of dates. Then, we use standard SQL functions to attach a number of attributes we'll use to create the 4-5-4 calendar.

Let's start with our base `dates` model, `dates.sql`:

(This intentionally leaves out a **ton** of otherwise useful attributes such as Week, Month, Quarter etc. to help us focus.)

{% raw %}
```sql
{{
    config(
        materialized = 'ephemeral'
    )
}}
with dates as
(
    -- we arbitrarily start on 1/1/2016 and end 53 weeks from now:
    {{ dbt_utils.date_spine(
        datepart="day",
        start_date="to_date('01/01/2016', 'mm/dd/yyyy')",
        end_date="dateadd(week, 53, current_date)"
       )
    }}
)
select
    d.calendar_date,
    date_trunc('week', d.calendar_date)::date as week_start_date,
    case 
        when day_of_week = 7 then d.calendar_date
        else dateadd('day', -1, week_start_date) 
    end as week_start_date_sun,
    dateadd('day', 6, date_trunc('week', d.calendar_date))::date as week_end_date,
    dateadd('day', 6, week_start_date_sun) as week_end_date_sat,
    date_part('month', d.calendar_date)::int as month_of_year,
    date_part('year', d.calendar_date)::int as year_number
from
    dates d
order by 1
```
{% endraw %}

### 4-5-4 Periods
Then using this `ephemeral` model (i.e. we chose to not materialize this as a table or view at the moment), we can calculate the 4-5-4 attributes in a separate **dbt** model (thus, separating the 2 models in 2 files for better organization and reusabilty).

`retail_calendar.sql`
{% raw %}
```sql
{{
    config(
        materialized = 'table'
    )
}}
with retail_year as 
(   -- get first first full week starting Sunday in February by year
    -- (week_end_date is a Sunday per our Snowflake config, 
    -- so this gets the first week *starting* Sunday)
    select 
        year_number as retail_year_number,
        min(week_end_date) as retail_year_start_date
    from 
        {{ ref('dates') }}
    where 
        month_of_year = 2 
    group by 1
),
retail_year_range as 
(
    select
        retail_year_number,
        retail_year_start_date,
        -- we compute the end of the 'retail' year 
        -- by shifting the next start date back one day 
        dateadd('day', -1, 
            lead(retail_year_start_date) over(order by retail_year_start_date)
            ) as retail_year_end_date
    from retail_year
),
retail_dates as 
(
    select
        d.calendar_date,
        d.year_number,
        m.retail_year_number,
        m.retail_year_start_date,
        m.retail_year_end_date,
        d.week_start_date,
        d.week_start_date_sun,
        -- we reset the weeks of the year starting 
        -- with the retail year start date
        dense_rank() 
            over(
                partition by m.retail_year_number 
                order by d.week_start_date_sun
                ) as retail_week_of_year 
    from
        {{ ref('dates') }} d
        join
        retail_year_range m on 
            d.calendar_date between m.retail_year_start_date and 
                                    m.retail_year_end_date
    
),
retail_periods as 
(
    select
        calendar_date,
        week_start_date_sun,
        retail_week_of_year,
        retail_week_of_year-1 as week_num,
        -- We count the weeks in a 12 week period
        -- and separate the 4-5-4 week sequences
        mod(week_num::float, 13) as w13num,
        case 
            when w13num between 0 and 3 then 1
            when w13num between 4 and 8 then 2
            when w13num between 9 and 12 then 3
        end as period_of_quarter,
        -- Chop weeks into 13 week retail quarters
        trunc(week_num/13) as qrt_num,
        (qrt_num * 3) + period_of_quarter as retail_period_number
    from
        retail_dates
)
select
    calendar_date,
    week_start_date_sun,
    retail_week_of_year, 
    dense_rank() 
        over(
            partition by retail_period_number 
            order by retail_week_of_year
            ) as retail_week_of_period,
    retail_period_number,
    qrt_num as retail_quarter_number,
    period_of_quarter as retail_period_of_quarter
from 
    retail_periods 
order by 1

```
{% endraw %}

This then hopefully leaves us with dates and weeks properly grouped into their respective 4-5-4 periods. In production, we'd also add the relevant retail holidays, which will leave for a future post.
Also, some retailers and corporate finance departments use other variations of this, such as 5-4-4, which you should be able to implement with minor adjustments using this approach.  
