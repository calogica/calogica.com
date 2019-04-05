---
title:  "Setting up a data pipeline using Snowflake's Snowpipes"
date:   2019-04-05 8:00AM
excerpt: "We take a closer look at setting up a data pipeline for file-based data sources using Snowflake's powerful Snowpipes feature."
categories: [sql, snowflake]
comments: true
---
{: .notice--info}
In this post, we look at the steps required to set up a data pipeline to ingest text based data files stored on **s3** into Snowflake using **Snowpipes**.

**Snowpipes** is one of the more unique and powerful, yet somewhat under-documented, or at least not much talked about features in Snowflake.

While we really like [**Fivetran**](https://www.fivetran.com) for handling data ingestion, there are cases where Fivetran's s3 connector doesn't provide quite the flexibility or peformance we're looking for. In particular, if you have a lot of files to ingest (e.g. from a process that generates many smaller files), the Fivetran s3 connector can be slow, to the point where it might be counter-productive.

So, what to do? 

If you're already using Snowflake, how about a data pipeline in Snowpipes?

Here are the steps to get going:

## 1) Set up a separate database

We like to set up a separate **database** in Snowflake for any source datasets that don't come in via Fivetran. (Also, we keep all source data outside our data warehouse database to begin with.)

For example, let's create a database called `etl`:

```sql
create database etl;
use etl;
```

## 2) Set up a schema to hold our source data

For the sake of this example, we'll load our data in the `src` schema:

```sql
create schema src;
```

## 3) Create a Table

Since we don't have the benefit of **Fivetran** creating our table, we need to create a **table** to hold our data.

Let's assume a fictional table called `my_source_table` with 2 columns:

```sql
create table src.my_source_table
(
	col_1 varchar,
	col_2 varchar
);
```

## 4) Create the File Format

The first real step is to create a `file format` that lets us control the type of file we want to ingest. 

Even if your data is in a simple **csv** file, it makes sense to explicitly control the file format options. Snowflake provides a host of file format options [here](https://docs.snowflake.net/manuals/sql-reference/sql/create-file-format.html).

For our example, we create a comma-delimited file format with a header that has values quoted in `"` and may have `null` fields encoded with the string `'null'`. (I mean, really, who does that?)

```sql
create or replace file format my_csv_format
  type = csv field_delimiter = ',' skip_header = 1
  field_optionally_enclosed_by = '"'
  null_if = ('NULL', 'null') 
  empty_field_as_null = true;
```

Let's check the existing file formats to make sure this got created:

```sql
show file formats;
```

## 5) Create an external stage pointing to your s3 location

A `stage`, in Snowflake terms, is a pointer to an internal or external (i.e. _your_) s3 location where your data files are stored - encoding not just the path, but also any authentication information. (More [here](https://docs.snowflake.net/manuals/sql-reference/sql/create-stage.html#external-stages).)


```sql
create or replace stage my_stage url='s3://my_bucket/key/key/'
  credentials=(aws_key_id='KEY' aws_secret_key='SECRET')
  file_format = my_csv_format;
```
Or use an IAM role:

```sql
create or replace stage my_stage url='s3://my_bucket/key/key/'
  credentials=(aws_role='aws_iam_role=arn:aws:iam::XXXXXXX:role/XXXX')
  file_format = my_csv_format;
```

Let's review all of our stages in this database:
```sql
show stages;
```

## 6) Review staged files and select data from the files 

Let's make sure security and file formats are both working as expected. The best way is to use the `list` feature to get a listing of files in our staging location. 

```sql
list @my_stage;
```

You can also query the raw files directly to make sure the delimiters are working as expected, although I wouldn't recommend using this as anything but a way to debug issues, and definitely not to read data files for production purposes. 
(More [here](https://docs.snowflake.net/manuals/user-guide/querying-stage.html).)

```sql
select t.$1, t.$2
from @my_stage (file_format => my_csv_format) t;
```

## 7) Test loading data into the table

Before we set up a **Snowpipe**, we should make sure we can actually import data from the files into the table we've set up.
Snowflake provides a few ways to limit the number of files we can copy to our table, which is especially helpful during testing if you have a lot of files.

For example, you can use a RegEx `pattern` to limit the number of files to load. (More [here](https://docs.snowflake.net/manuals/user-guide/data-load-s3-copy.html).)

```sql
copy into src.my_source_table
  from @my_stage
  pattern='.*sales.*.csv';
;
```

If this succeeded, don't forget to clear out your table before setting up the Snowpipe. 
```sql
truncate src.my_source_table;
```

## 8) Create the Snowpipe

Ultimately, we want to automate the copy command so we need a Snowpipe that will make this a bit easier to manage.

(More info on [Snowpipes](https://docs.snowflake.net/manuals/sql-reference/sql/create-pipe.html).)

```sql
create pipe if not exists my_pipe as
copy into src.my_source_table from @my_stage;
```

Confirm that this worked as expected:
```SQL
show pipes;
```

## 9) Verify your table is empty 

Before kicking off the **Snowpipe**, let's make sure we don't have previously loaded data in our table.

```sql
select count(*) from src.my_source_table;
```

## 10) Force a pipe refresh

Using the `alter pipe` command and the `refresh` option we can force a **Snowpipe** to send any files from its associated stage to an ingestion queue. You can read more [here](https://docs.snowflake.net/manuals/sql-reference/sql/alter-pipe.html).

```sql
alter pipe my_pipe refresh;
```


Since this sends files to a queue, we'll wait a bit for Snowflake to process the queue of staged files, then we'll verify your post-load row row count.

We can use the built-in `pipe_status` command to check on our pipe's status and how many files are current in the queue.

```sql
select system$pipe_status('my_pipe');
```

If files are no longer queued, check to make sure your table has the expected number of records.

```sql
select count(*) from src.my_source_table;
```

Hopefully this has given you some insights into using Snowpipes for data pipelines that can't be handled by your favorite data pipeline Saas vendor. 

Snowflake is continuing to build out support for Snowpipes, so stay tuned for updates in this space!