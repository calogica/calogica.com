---
title:  "Bulk-loading data from pandas DataFrames to Snowflake"
date:   2019-06-12 8:00AM
excerpt: "We examine how to bulk-load the contents of a pandas DataFrame to a Snowflake table using the copy command."
categories: [sql, snowflake, python]
comments: true
---
{: .notice--info}
In this post, we look at options for loading the contents of a pandas DataFrame to a table in Snowflake directly from Python, using the copy command for scalability.

## Snowflake as part of the Data Science Workflow

As the recent Snowflake [Summit](https://www.snowflake.com/summit/), one of the questions we got to discuss with Snowflake product managers was how to better integrate Snowflake in a data science workflow.

Often this workflow requires:
1. Sourcing data (often a training dataset for a machine learning project) from our Snowflake data warehouse
2. Manipulating this data in a pandas DataFrame using statistical techniques not available in Snowflake, or using this data as input to train a machine learning model
3. Loading the output of this model (e.g. a dataset scored using the trained ML model) back into Snowflake by copying a `.csv` file to an S3 bucket, then creating a Snowpipe or other data pipeline process to read that file into a Snowflake destination table.

Much of this work is boilerplate, and once you've done this once it's pretty boring. Thus, an excellent use case for a library to handle this.

While we're still waiting for Snowflake to come out with a fully Snowflake-aware version of pandas (we, so far, unsuccessfully pitched this as [SnowPandas&trade;](https://www.youtube.com/watch?v=hhSUandetlE) to the product team), let's take a look at quick and dirty implementation of the read/load steps of the workflow process from above.

## Reading data from Snowflake in Python

### Import Libraries
We'll make use of a couple of popular packages in Python (3.6+) for this project, so let's make we `pip install` and import them first:

```python
import os

import pandas as pd

import sqlalchemy
from sqlalchemy import create_engine
from snowflake.sqlalchemy import URL
```
We're using SQLAlchemy here in conjunction with the `snowflake.sqlalchemy` library, which we install via `pip install --upgrade snowflake-sqlalchemy`. For more information, check out the Snowflake docs on [snowflake-sqlalchemy](https://docs.snowflake.net/manuals/user-guide/sqlalchemy.html#installing-snowflake-sqlalchemy).

### Creating the Engine

To use SQLAlchemy to connect to Snowflake, we have to first create an `engine` object with the correct connection parameters. This `engine` doesn't have an open connection or uses any Snowflake resources *until* we explicitly call `connect()`, or run queries against it, as we'll see in a bit.

```python
engine = create_engine(URL(
        account=os.getenv("SNOWFLAKE_ACCOUNT"),
        user=os.getenv("SNOWFLAKE_USER"),
        password=os.getenv("SNOWFLAKE_PASSWORD"),
        role="<role>",
        warehouse="<warehouse>",
        database="<database>",
        schema="<schema>"
    ))
```

Ideally, our security credentials in this step come from environment variables, or some other more secure method than leaving those readable in our script. 

### Connect 

Connecting to Snowflake, or really any database SQLAlchemy supports, is as easy as the snippet below. We assume we have our source data, in this case a pre-processed table of training data `training_data` for our model (ideally built using [dbt](https://getdbt.com)).

Note that we're using our `engine` in a Python context manager (`with`) here to make sure the connection gets properly closed and disposed after we're done reading.

Also, we're making use of `pandas` built-in [read_sql_query](https://pandas.pydata.org/pandas-docs/stable/reference/api/pandas.read_sql_query.html) method, which requires a `connection` object and happily accepts our connected SQLAlchemy engine object passed to it in the context.

```sql
sql = "select * from training_data"
```

```python
with engine.connect() as conn:
        df = pd.read_sql_query(sql, con=conn)
```

One caveat is that while `timestamps` columns in Snowflake tables correctly show up as `datetime64` columns in the resulting DataFrame, `date` columns transfer as `object`, so we'll want to convert them to proper pandas timestamps.

In our example, we assume any column ending in `_date` is a date column.

```python
for c in df.columns:
    if c.endswith("_date"):
        df[c] = pd.to_datetime(df[c])
```

This will help us later when we **create** our target table programmatically.

### Do Science

Now that have our training data in a nice DataFrame, we can pass it to other processing functions or models as usual.
 
For example, some pseudo code for illustration:

```python
def fancy_machine_learning_model(data):

    train_data, test_data = train_test_split(data)

    trained_model = train(train_data)

    trained_model.evaluate(test_data)

    predicted_data = trained_model.predict(data)

    return predicted_data
```

```python
df_predicted = fancy_machine_learning_model(df)
```

### Upload to Snowflake

Now that we have a scored dataset with our predictions, we'd like to load this back into Snowflake.

Let's think of the steps normally required to do that:
1. **Save** the contents of the DataFrame to a file
2. **Upload** the file to a location Snowflake can access to load, e.g. a stage in S3
3. **Create** the target table if necessary, or **truncate** the target table if necessary
3. Run a `copy` command in Snowflake to **load** the data

We could imagine wrapping these steps in a reusable function, like so:

(We'll go over the pertinent bits below)

```python        
def upload_to_snowflake(data_frame, 
                        engine, 
                        table_name, 
                        truncate=True, 
                        create=False):

    file_name = f"{table_name}.csv"
    file_path = os.path.abspath(file_name)
    data_frame.to_csv(file_path, index=False, header=False)

    with engine.connect() as con:

        if create:
            data_frame.head(0).to_sql(name=table_name, 
                                      con=con, 
                                      if_exists="replace", 
                                      index=False)
        if truncate:
            con.execute(f"truncate table {table_name}")

        con.execute(f"put file://{file_path}* @%{table_name}")
        con.execute(f"copy into {table_name}") 
```

First we save our data locally. Note that we're **not** saving the column headers or the index column. Column headers will interfere with the `copy` command later.

```python
...
file_name = f"{table_name}.csv"
file_path = os.path.abspath(file_name)
data_frame.to_csv(file_path, index=False, header=False)
...
```
For larger datasets, we'd explore other more scalable options here, such as [dask](https://dask.org/). 

Next, we once again wrap our connection in a context manager:
```python
...
with engine.connect() as con:
...
```

If we need to create the target table (and your use case may vary wildly here), we can make use of pandas `to_sql` method that has the option to create tables on a connection (provided the user's permissions allow it). 

However, note that we do **not** want to use `to_sql` to actually upload any data. The `to_sql` method uses `insert` statements to insert rows of data. Even in it's bulk mode, it will send one line of `values` per row in the dataframe. That's fine for smaller DataFrames, but doesn't scale well.

So, instead, we use a header-only DataFrame, via `.head(0)` to force the creation of an empty table. In this example, we also specify to replace the table if it already exists.
Pandas, via SQLAlchemy, will try to match the DataFrame's data types with corresponding types in Snowflake. For the most part, this will be fine, but we may want to verify the target table looks as expected. 

```python
...
if create:
    data_frame.head(0).to_sql(name=table_name, 
                                con=con, 
                                if_exists="replace", 
                                index=False)
...
```

In the event that we simply want to `truncate` the target table, we can also run arbitrary SQL statements on our connection, such as:

```python
if truncate:
    con.execute(f"truncate table {table_name}")
```

Next, we use a Snowflake `internal` stage to hold our data in prep for the `copy` operation, using the handy [put](https://docs.snowflake.net/manuals/sql-reference/sql/put.html) command:

```python
con.execute(f"put file://{file_path}* @%{table_name}")
```
Depending on operating system, `put` will require different path arguments, so it's worth reading through the docs. 
In our example, we're uploading our file to an internal stage specific to our target table, denoted by the `@%` option.

Also, note that `put` auto-compresses files by default before uploading and supports threaded uploads. For example, from the docs:

> Larger files are automatically split into chunks, staged concurrently and reassembled in the target stage. A single thread can upload multiple chunks.

For our example, we'll use the default of `4` threads.

We could also load to and from an `external` stage, such as our own S3 bucket. In that case, we'd have to resort to using `boto3` or another library to upload the file to S3, rather than the `put` command.

Lastly, we execute a simple `copy` command against our target table. Since we've loaded our file to a table stage, no other options are necessary in this case.

```python
con.execute(f"copy into {table_name}") 
```

Note that Snowflake does *not* copy the same staged file more than once unless we truncate the table, making this process idempotent. If we wanted to append multiple versions or batches of this data, we would need to change our file name accordingly before the `put` operation.

### How will you use it? 

There are many other use cases and scenarios for how to integrate Snowflake into your data science pipelines. Hopefully this post sparked some ideas and helps speed up your data science workflows. Looking forward to hearing your ideas and feedback!