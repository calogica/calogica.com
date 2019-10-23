---
title:  "How to Backup Snowflake Data to S3"
date:   2019-09-09 8:00AM
excerpt: "How to backup a Snowflake database to S3."
categories: [sql, snowflake, dbt]
toc: true
toc_label: "What we'll cover"
toc_icon: "snowflake"
toc_sticky: true
---

{: .notice--info}
In this post, we take a look at how to back up data in your Snowflake account to an external s3 location.

Snowflake has amazing built-in backup functionality, called [Time Travel](https://docs.snowflake.net/manuals/user-guide/data-time-travel.html) that lets you access data *as of* a certain date. In addition, Snowflake protects your data in the event of a system failure or other catastrophic event with its [Fail-safe](https://docs.snowflake.net/manuals/user-guide/data-failsafe.html) feature which allows Snowflake support to restore data for you during the Fail-safe window.

In addition, Snowflake has a near magical ability to `undrop` objects that were dropped, which is a direct result of its immutable architecture.

However, Time Travel is limited to 1 day for Standard Edition (up to 90 days for Enterprise Edition), while Fail-safe adds 7 days of peace of mind. 
So, while both Time Travel and Fail-safe are convenient and storage-efficient means of backing up your data, some Snowflake customers, especially those using Standard Edition, may want to back up their data using alternate means.

One easy way is to simply unload your data to an AWS S3 bucket, using Snowflake's built-in `copy into` [command](https://docs.snowflake.net/manuals/sql-reference/sql/copy-into-location.html). Let's explore how to automate this using everyone's favorite data transformation library `dbt` for a number of databases and schemas. (See more on dbt [here](https://www.getdbt.com).)

## S3 External Stages
Snowflake can access external (i.e. in *your* AWS account, and not within Snowflake's AWS environment) S3 buckets for both read and write operations. The easiest way to take advantage of that is to create an `external stage` in Snowflake to encapsulate a few things. (We've used stages in a [previous post](/sql/snowflake/2019/04/04/snowpipes.html) on loading data using Snowpipes.)

### Create S3 Stage

Let's create a dedicated stage for our backups:

```sql
create or replace stage backup_stage url='s3://my_s3_bucket/snowflake_backup'
credentials=(aws_key_id='<AWS_KEY_ID>', aws_secret_key='<AWS_SECRET_KEY>')
file_format=(type=csv compression='GZIP' field_optionally_enclosed_by='"', skip_header=1)
```
- Note that we're specifying the `csv` file format, along with `gzip` compression as part of the stage definition. Your business case will vary here, and `parquet` or other format types may be better choices for you.
- The `skip_header` option here only applies to reading data from this stage, not to unloading data. We **will* specify during our unload process that we want to save table headers with our backups. However, for restore purposes, we want to skip headers. You can leave this option out if you'd rather get the relevant table headers back as the first row when you read from this external stages later.

Find more informtion on creating stages [here](https://docs.snowflake.net/manuals/sql-reference/sql/create-stage.html#external-stages).

### Copy into Stage

We can now `copy into` our external stage from any Snowflake table:

```sql
copy into @backup_stage/my_database/my_schema/my_table/data_
from my_database.my_schema.my_table
header = true
overwrite = true
max_file_size = 104857600
```

Let's quickly talk about what's going on here:
- We copying *from* a table *into* our external S3 stage, which uses the compressed format specified earlier.
- By naming the output file `data_`, with no other related options, we specify that we want Snowflake to create **multiple** files, all starting with `data_*`, which allows Snowflake to run this command in parallel in our virtual warehouse (parallel runs are limited by the size of the virtual warehouse).
- We want the exported files to include table **headers**
- We're **overwriting** any existing files at that location
- We specified an **upper limit** of `100MB` per file (`100*1,024^2 = 104,857,600 bytes`) (See Snowflake's [General File Sizing Recommendations](https://docs.snowflake.net/manuals/user-guide/data-load-considerations-prepare.html#general-file-sizing-recommendations) for more.)

There are many more options to be explored, see the `copy into` [docs for help](https://docs.snowflake.net/manuals/sql-reference/sql/copy-into-location.html).


## Automation via dbt

So, this helped backing up *one* table. However, we likely have dozens or hundreds more, across various schemas and databases.
Thankfully, `dbt` allows us to script generation of SQL commands using `jinja2`

What we really want is a flexible `macro` that will build the `copy into` command for a given database/schema/table combination.

For example:
{% raw %}
```jinja
{% macro get_backup_table_command(table, day_of_month) %}
{% set backup_key -%}
day_{{ day_of_month }}/{{ table.database.lower() }}/{{ table.schema.lower() }}/{{ table.name.lower() }}/data_
{%- endset %}
copy into @backup_stage/{{ backup_key }}
from {{ table.database }}.{{ table.schema }}."{{ table.name.upper() }}"
header = true
overwrite = true
max_file_size = 1073741824;
{% endmacro %}
```
{% endraw %}

In this case, we're creating the `copy into` statement that will back up our given table into an AWS S3 key that looks something like this:

`s3://my_s3_bucket/snowflake_backup/day_09/dw/shop/dim_product/`

(We're upper-casing the table name in quotes in the `from` clause to guard against reserved words in table names, e.g. such as "ORDER" in the case of Fivetran-sourced Shopify data. 😒)

Using this macro, we could loop through a list of database, schemas and tables and execute backup statements like this in sequence.

Let's see how that would look in a `dbt` macro.
- The key components here is the `backups` *dictionary* that specifies a `list` of schemas for each database `key`. In this example, we're backing up 2 databases (`"RAW"` and `"DW"`) with several schemas in each.
- In this case we're looking to backup *all* tables in each specified schema, so we're using the handy `get_tables_by_prefix` macro from the `dbt_utils` package to get a list of `relations` (i.e. tables) that match our filter. Note that we can exclude tables as well, such as any metadata tables (exluding anything starting with  `FIVETRAN_%`).
- In this example, we chose to keep up to 31 days of backups, one for each day of the month. Once the momth is over, we simply roll over and start overwriting backups. This is a **very simple** backup scheme, so please make sure this modify this suit your business needs. This is just meant to illustrate that we have access to *some* of Python's date formatting functionality via Jinja to create SQL statements. 
- Since we ultimately want to operationalize this backup process using a `dbt run-operation`, we wrap our code in a `statement` which allows us to use a `run-operation` to execute SQL that does not exclude a `select` statement.

{% raw %}
```jinja
{% macro backup_to_s3() %}

    {%- call statement('backup', fetch_result=true, auto_begin=true) -%}

        {% set backups =
            {
                "RAW":
                    ["APP_DATA",
                    "FACEBOOK", 
                    "ADWORDS", 
                    "SHOPIFY", 
                    "ZENDESK"],
                "DW":
                    ["CUSTOMER", 
                    "SHOP",
                    "FINANCE"]
            }
        %}

        {% set day_of_month = run_started_at.strftime("%d") %}
        
        {{ log('Backing up for Day ' ~ day_of_month, info = true) }}

        {% for database, schemas in backups.items() %}
        
            {% for schema in schemas %}
        
                {{ log('Getting tables in schema ' ~ schema ~ '...', info = true) }}

                {% set tables = dbt_utils.get_tables_by_prefix(schema.upper(), '', exclude='FIVETRAN_%', database=database) %}

                {% for table in tables %}
                    {{ log('Backing up ' ~ table.name ~ '...', info = true) }}
                    {% set backup_table_command = get_backup_table_command(table, day_of_month) %}
                    {{ backup_table_command }}
                {% endfor %}
        
            {% endfor %}
        
        {% endfor %}

    {%- endcall -%}

{%- endmacro -%}
```
{% endraw %}

We can then call this from the command line like so,

```bash
dbt run-operation backup_to_s3
```
which we can then automate in our CI/CD environment of choice, such as **dbt Cloud** or **Gitlab**.

## Data Recovery

So, how do we use these backup files if we want to recover data from a previous backup?
A great thing about Snowflake external stages is that we can simply read from them, using the same stage definition we've used for unloading data to them.
For example, we could pick a specific table (`my_table`) and day (`day_09`), and read from the backed up data like so:

```sql
select 
    $1 as id, 
    to_timestamp($2, 'yyyy-mm-dd hh24:mi:ss.ff Z') as ts 
from 
    @backup_stage/day_09/my_db/my_schema/my_table/
```

Which would result in output like this:
```
| ID      | TS                            |
|---------|-------------------------------|
| 9147485 | 2019-10-09 15:06:43.627000000 |
| 9147484 | 2019-10-09 15:06:36.230000000 |
| 9147483 | 2019-10-09 15:06:28.263000000 |
| 9147482 | 2019-10-09 15:06:09.158000000 |
| 9147481 | 2019-10-09 15:06:02.582000000 |
```

Consequently, we can create a table in Snowflake with the recovered data:

```sql
create table my_table_recovered as (

    select 
        $1 as id, 
        to_timestamp($2, 'yyyy-mm-dd hh24:mi:ss.ff Z') as ts 
    from 
        @backup_stage/day_09/my_db/my_schema/my_table/

)
```

If we're happy with the recovered data, we can then swap the recovered data with the current ("bad") data,

```sql
alter table my_table rename to my_table_bad;
alter table my_table_recovered rename to my_table;
```

Then at a future point, we could delete the "bad" table:
```sql
drop table my_table_bad;
```

Hopefully, this gave you some idea of how to extend the built-in Snowflake recovery features like Time Travel and Fail-safe with a few alternatives. 

As always, please let us know if you have any feedback or comments!