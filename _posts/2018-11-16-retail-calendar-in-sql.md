---
title:  "Creating a 4-5-4 Retail Calendar using SQL and dbt (UPDATED)"
date:   2018-11-15 7:00PM
excerpt: "Using SQL and dbt, we create a date dimension table using a retail/merchandising calendar known as a 4-5-4 calendar."
categories: [sql, dbt]
comments: true
---

{: .notice--info}
Note: this post has been updated to include handling of 53-week years and refactors some of the calendar logic into a dbt macro for flexibility. Thanks for everyone's feedback!

Working with clients in retail eCommerce, we're often asked to analyze transactions along a type of fiscal calendar optimized for retail and merchandising. This is commonly known as the "4-5-4 Calendar", because it groups weeks (Sun-Sat) into periods of 4, 5 and 4 weeks lengths.

From the **National Retail Federation** [https://nrf.com/resources/4-5-4-calendar](https://nrf.com/resources/4-5-4-calendar):
> The 4-5-4 calendar is a guide for retailers that ensures sales comparability between years by dividing the year into months based on a 4 weeks – 5 weeks – 4 weeks format. The layout of the calendar lines up holidays and ensures the same number of Saturdays and Sundays in comparable months. Hence, like days are compared to like days for sales reporting purposes. The 4-5-4 Calendar also establishes Sales Release dates, which have historically been on the first Thursday following the month’s end. In recent years, however, as the flow of information has improved, more companies are releasing sales data earlier in the week.

For example, the retail year 2019 will start on February 3, 2019 and group the first few weeks of the year like so:


!["4-5-4 Calendar"](/assets/plots/cal454.png "4-5-4 Calendar")

Also, some years have 53 weeks. Again, from the NRF:
> Dividing the retail calendar into 52 weeks of seven days each, or 364 days, leaves an extra day each year to be accounted for. As a result, every five to six years a week is added to the fiscal calendar. This anomaly has most recently occurred in FY12 and FY17 and will occur in FY23.

E.g.:
!["4-5-4 Calendar (W53)"](/assets/plots/cal454_w53.png "4-5-4 Calendar (W53)")

Teasing apart some of this, we draw up the following rules:
- Retail weeks start on Sunday
- Retail years end in January
- The exact year end date is determined by which Saturday lies closest to January 31. It looks like we can follow the rule outlined in this [Wikipedia article](https://en.wikipedia.org/wiki/4%E2%80%934%E2%80%935_calendar#Saturday_nearest_the_end_of_month).
- If a year has 53 weeks, Week 53 is part of period 12 


Let's explore how to create a 4-5-4 calendar using SQL for our data warehouse (we're using Snowflake here, but this should be adaptable to Redshift or other platforms), and how to use **dbt** (our favorite data transformation tool) to make that easier.

Side note: if you're not already using **[dbt](https://www.getdbt.com/){:target="_blank"}.** to manage your data transformations, we highly recommend you take a look at it for your Redshift, Snowflake or Big Query data warehouse projects. It's been an indispensable tool for us over the last 18 months! 

### Date Dimension
As good data warehouse practitioners, we of course already have a date dimension that contains past and future dates relevant to our business, along with descriptive attributes (Day of Week etc) and groupings (Week, Month, Quarter, Year) that help us create actionable analysis. 

As an extension, it's quite common to add attributes such as holidays, as well non-calendar based attributes such as fiscal calendar based groupings.

But, if we're just starting out, we need to create a date dimension for our project. We will use the helpful `date_spine` macro from the `dbt_utils` package to create a sequence of dates. Then, we use standard SQL functions to attach a number of attributes we'll use to create the 4-5-4 calendar.

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
    date_trunc('month', d.calendar_date)::date as month_start_date,
    {{ dbt_utils.last_day('d.calendar_date', 'month') }} as month_end_date,
    date_part('year', d.calendar_date)::int as year_number
from
    dates d
order by 1
```
{% endraw %}

### Fiscal Calendar
As we've seen, fiscal calendars usually, at a minimum, differ from the traditional calendar by their start and end dates. On top of that, weeks are often grouped into comparable groupings. 

Since the 4-5-4 calendar is just _one_ of the versions of fiscal calendars we might encounter, we'll encapsulate some of the core logic behind determining fiscal year start and end dates in a **dbt** macro for extensability.

We'll parameterize the macro to allow us to chose the _month_ when our fiscal calendar ends, and the _weekday_ when our fiscal weeks start (using Sunday=0). That way, we can design fiscal weeks that are independent of the calendar weeks we created earlier in our date dimension. 


`fiscal_year_dates.sql`

{% raw %}
```sql
{% macro fiscal_year_dates(year_end_month, week_start_day=0, shift_year=1) %}
-- this gets all the dates within a fiscal year 
-- determined by the given year-end-month
-- ending on the saturday closest to that month's end date
with year_month_end as
(
    select
        -- This, while slightly lazy, accounts for the fact
        -- that most fiscal years end in a month following the calendar year
       d.year_number-{{ shift_year }} as fiscal_year_number,
       d.month_end_date
    from
        {{ ref('dates') }} d
    where
        d.month_of_year = {{ year_end_month }}
    group by 1,2
),
weeks as 
(
    select
        d.calendar_date as week_start_date,
        dateadd('d', 6, d.calendar_date) as week_end_date
    from
        {{ ref('dates') }} d
    where 
        date_part('dow', d.calendar_date) = {{ week_start_day }}
),
-- get all the weeks that start in the month the year ends
year_week_ends as
(
    select
        d.year_number-{{ shift_year }} as fiscal_year_number,
        d.week_end_date
    from
        weeks d
    where
        date_part('month', d.week_start_date) = {{ year_end_month }}
    group by 1,2
),
-- then calculate which Saturday is closest to month end
weeks_at_month_end as
(
    select
        d.fiscal_year_number,
        d.week_end_date,
        m.month_end_date,
        rank() over
            (partition by d.fiscal_year_number
                order by
                abs(datediff('d', d.week_end_date, m.month_end_date))

            ) as closest_to_month_end
    from
        year_week_ends d
        join
        year_month_end m on d.fiscal_year_number = m.fiscal_year_number
),
fiscal_year_range as 
(
    select
        fiscal_year_number,
        dateadd('day', 1, 
            lag(week_end_date) over(order by week_end_date)
        ) as fiscal_year_start_date,
        week_end_date as fiscal_year_end_date
    from
        weeks_at_month_end
    where closest_to_month_end = 1
),
fiscal_year_dates as (
    select
        d.calendar_date,
        m.fiscal_year_number,
        m.fiscal_year_start_date,
        m.fiscal_year_end_date,
        w.week_start_date,
        w.week_end_date,
        -- we reset the weeks of the year starting with the merch year start date
        dense_rank() 
            over(
                partition by m.fiscal_year_number 
                order by w.week_start_date
                ) as fiscal_week_of_year 
    from
        {{ ref('dates') }} d
        join
        fiscal_year_range m on d.calendar_date between m.fiscal_year_start_date and m.fiscal_year_end_date
        join
        weeks w on d.calendar_date between w.week_start_date and w.week_end_date
),
{% endmacro %}
```
{% endraw %}

In this macro, we essentially construct a calendar grouped into a fiscal year given by our end month (and end date logic discussed earlier) and fiscal weeks starting on our specified weekday.

Now, to group these weeks into the fiscal periods required for our retail calendar, we use this macro in a downstream model.

### 4-5-4 Periods
Using both our _ephemeral_ `dates` model (i.e. we chose to not materialize this as a table or view at the moment) and the `fiscal_year_dates` macro, we can calculate the 4-5-4 attributes in a separate **dbt** model (thus, separating the 2 models in 2 files for better organization and reusabilty).

`retail_calendar.sql`

{% raw %}
```sql
{{
    config(
        materialized = 'table'
    )
}}
-- year ends in January = 1
-- weeks start on Sunday = 0
{{ fiscal_year_dates(1, 0) }}
retail_periods as 
(
    select
        calendar_date,
        fiscal_year_number as retail_year_number,
        week_start_date,
        week_end_date,
        fiscal_week_of_year as retail_week_of_year,
        fiscal_week_of_year-1 as week_num,
        -- We count the weeks in a 13 week period
        -- and separate the 4-5-4 week sequences
        mod(week_num::float, 13) as w13_number,
        -- Chop weeks into 13 week merch quarters
        least(trunc(week_num/13),3) as quarter_number,
        case 
            -- we move week 53 into the 3rd period of the quarter
            when fiscal_week_of_year = 53 then 3
            when w13_number between 0 and 3 then 1
            when w13_number between 4 and 8 then 2
            when w13_number between 9 and 12 then 3
        end as period_of_quarter,
        (quarter_number * 3) + period_of_quarter as retail_period_number
    from
        fiscal_year_dates
)
select
    calendar_date,
    retail_year_number,
    week_start_date,
    week_end_date,
    retail_week_of_year, 
    dense_rank() over(
        partition by retail_year_number, retail_period_number 
        order by retail_week_of_year) as retail_week_of_period,
    retail_period_number,
    quarter_number+1 as retail_quarter_number,
    period_of_quarter as retail_period_of_quarter
from 
    retail_periods 
order by 1,2

```
{% endraw %}

This then hopefully leaves us with dates and weeks properly grouped into their respective 4-5-4 periods. In production, we'd also add the relevant retail holidays, which will leave for a future post.

Let's take a quick look at the output of this in **Tableau**, which matches up nicely with the "official" NRF calendar we saw earlier:
!["4-5-4 Calendar in Tableau"](/assets/plots/cal454_tableau.png "4-5-4 Calendar in Tableau")

We'll also check to make sure years with 53 weeks, such as 2017, roll up that week into period 12 as expected.
!["4-5-4 Calendar in Tableau (W53)"](/assets/plots/cal454_tableau_w53.png "4-5-4 Calendar in Tableau (W53")

Note: some retailers rectroactively restate years with 53 weeks. You can see an example of the effect of that on the NRF website [here](https://6a83cd4f6d8a17c1b6dd-0490b3ba35823e24e2c50ce7533598b0.ssl.cf1.rackcdn.com/454%20Calendars/3%20Year%20Calendar%2017-19%20with%202017%20Restated.pdf), where 2017 has been restated with only 52 weeks.

Lastly, some retailers and corporate finance departments use other variations of this, such as 5-4-4, which you should be able to implement with minor adjustments using this approach.  
