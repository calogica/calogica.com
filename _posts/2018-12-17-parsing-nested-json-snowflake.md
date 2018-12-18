---
title:  "Parsing Nested JSON Dictionaries in SQL - Snowflake Edition"
date:   2018-12-17 8:00AM
excerpt: "How to parse nested dictionaries in Snowflake table columns using SQL"
categories: [sql]
comments: true
---
Over the last couple of months working with clients, we've been working with a few new datasets containing nested JSON.
In many cases, clients are looking to us to pre-process this data in Python or R to flatten out these nested structures into tabular data before loading to a data warehouse platform, such as Snowflake.

However, given the powerful (if under-documented) JSON features of Snowflake, we can often avoid a more complex Python-based processing pipeline, and query JSON data directly in our ELT pipelines (for example, as part of a **[dbt](https://www.getdbt.com/){:target="_blank"}** project).

Let's take an example dataset and explore 3 use cases for JSON manipulation in Snowflake:
- How to simple extract values from single-level JSON if we know the name of the keys ahead of time (sort of as a warm up)
- How to extract values from known JSON keys that are nested one or more levels deep (fun!)
- How to extract the keys _and_ values from JSON dictionaries, even nested ones

### Getting the Data

For the examples below, we'll assume the following data, which we get straight from the [Snowflake support documentation](https://support.snowflake.net/s/article/json-data-parsing-in-snowflake).


```json
{  
   "root":[  
      {  
         "kind":"person",
         "fullName":"John Doe",
         "age":22,
         "gender":"Male",
         "phoneNumber":{  
            "areaCode":"206",
            "number":"1234567"
         },
         "children":[  
            {  
               "name":"Jane",
               "gender":"Female",
               "age":"6"
            },
            {  
               "name":"John",
               "gender":"Male",
               "age":"15"
            }
         ],
         "citiesLived":[  
            {  
               "place":"Seattle",
               "yearsLived":[  
                  "1995"
               ]
            },
            {  
               "place":"Stockholm",
               "yearsLived":[  
                  "2005"
               ]
            }
         ]
      },
      {  
         "kind":"person",
         "fullName":"Mike Jones",
         "age":35,
         "gender":"Male",
         "phoneNumber":{  
            "areaCode":"622",
            "number":"1567845"
         },
         "children":[  
            {  
               "name":"Earl",
               "gender":"Male",
               "age":"10"
            },
            {  
               "name":"Sam",
               "gender":"Male",
               "age":"6"
            },
            {  
               "name":"Kit",
               "gender":"Male",
               "age":"8"
            }
         ],
         "citiesLived":[  
            {  
               "place":"Los Angeles",
               "yearsLived":[  
                  "1989",
                  "1993",
                  "1998",
                  "2002"
               ]
            },
            {  
               "place":"Washington DC",
               "yearsLived":[  
                  "1990",
                  "1993",
                  "1998",
                  "2008"
               ]
            },
            {  
               "place":"Portland",
               "yearsLived":[  
                  "1993",
                  "1998",
                  "2003",
                  "2005"
               ]
            },
            {  
               "place":"Austin",
               "yearsLived":[  
                  "1973",
                  "1998",
                  "2001",
                  "2005"
               ]
            }
         ]
      },
      {  
         "kind":"person",
         "fullName":"Anna Karenina",
         "age":45,
         "gender":"Female",
         "phoneNumber":{  
            "areaCode":"425",
            "number":"1984783"
         },
         "citiesLived":[  
            {  
               "place":"Stockholm",
               "yearsLived":[  
                  "1992",
                  "1998",
                  "2000",
                  "2010"
               ]
            },
            {  
               "place":"Russia",
               "yearsLived":[  
                  "1998",
                  "2001",
                  "2005"
               ]
            },
            {  
               "place":"Austin",
               "yearsLived":[  
                  "1995",
                  "1999"
               ]
            }
         ]
      }
   ]
}
```
(Download [json_sample_data2.json](assets/data/json_sample_data2.json))

Notice how this data actually includes 3 records for persons, the places they lived in during one or more years and their children, if any.


We'll upload this data file to Snowflake using the `SnowSQL` command line utlity, which creates a gzip compressed copy of our source file from above in the `@~/json/` user directory, as `json_sample_data2.json.gz`.

```
json_sample_data2.json_c.gz(0.00MB): [##########] 100.00% Done (0.435s, 0.00MB/s).
+------------------------+---------------------------+-------------+-------------+--------------------+--------------------+----------+---------+
| source                 | target                    | source_size | target_size | source_compression | target_compression | status   | message |
|------------------------+---------------------------+-------------+-------------+--------------------+--------------------+----------+---------|
| json_sample_data2.json | json_sample_data2.json.gz |        3342 |         507 | NONE               | GZIP               | UPLOADED |         |
+------------------------+---------------------------+-------------+-------------+--------------------+--------------------+----------+---------+
1 Row(s) produced. Time Elapsed: 1.573s
``` 

Then, following along with the instructions from the link above, we create the `json` file format:
```sql
create or replace file format json type = 'json';
```

Now we can query `json_sample_data2.json.gz` using Snowflake's SQL extensions for querying json, to extract the values encapsulated in the `root` element and flatten out each json record into a separate row:
```sql
select 
    t.value 
from 
    @~/json/json_sample_data2.json.gz (file_format => 'json') as S, 
    table(flatten(S.$1,'root')) t
```
Which should look like this:
![json_data](/assets/images/json_data_1.png)

Note the use of the `flatten` keyword, which we'll come back to in a bit.

To make accessing this data a little easier for the following examples, we'll load this data from the file into a (temp) table:
```sql
create temp table json_temp as
select t.value as json_data
from 
    @~/json/json_sample_data2.json.gz (file_format => 'json') as S, 
    table(flatten(S.$1,'root')) t
```

## Use Case 1: Extract Values When Keys Are Known:
Let's look at our first query example, where we assume that we _know_ the keys in the json dictionary we're interested in, and we simply want to create columns for each one.
In this case, let's get the *name*, *age*, *gender* and *phone number* for each person, using the `:` syntax to access dictionary key values:

```sql
select
    d.json_data:fullName::varchar as full_name,
    d.json_data:age::int as age,
    d.json_data:gender::varchar as gender,
    d.json_data:phoneNumber:areaCode::varchar as area_code,
    d.json_data:phoneNumber:number::varchar as phone_number
from
    json_temp d
```
This returns:
```
| FULL_NAME     | AGE | GENDER | AREA_CODE | PHONE_NUMBER |
|---------------|-----|--------|-----------|--------------|
| John Doe      | 22  | Male   | 206       | 1234567      |
| Mike Jones    | 35  | Male   | 622       | 1567845      |
| Anna Karenina | 45  | Female | 425       | 1984783      |
```
So far so good!

We also saw that some of the records include nested information about their children. Before we dive into that in more detail, let's just count how many children, if any, each person has:
```sql
select
    d.json_data:fullName::varchar as full_name,
    coalesce(
        array_size(d.json_data:children)
            , 0) as number_of_children
from
    json_temp d
```
We notice that `Anna Karenina` has no children. (Apparently, we are not following the [book](https://en.wikipedia.org/wiki/Anna_Karenina#Main_characters) here...)
```
| FULL_NAME     | NUMBER_OF_CHILDREN |
|---------------|--------------------|
| John Doe      | 2                  |
| Mike Jones    | 3                  |
| Anna Karenina | 0                  |
```


## Use Case 2: Extract Values From Nested Dictionaries


### One Level
Let's extract values from the nested `children` dictionary, given that we know the key names we're interested in.
We do this using the `lateral flatten` function, which expands each record in the `children` dictionary into a row, so that each person is shown along with each of their children's records.

We then extract the values for the `name`, `gender` and `age` keys.
There is also a handy built-in `index` that we can use to keep track of each child record.

Notice that we also specify the `outer` parameter in the `flatten` function to be `true`, which makes sure that we don't drop the childless `Anna Karenina` record.

```sql
select
    d.json_data:fullName::varchar as full_name,
    c.index as child_idx,
    c.value:name::varchar as child_name,
    c.value:gender::varchar as child_gender,
    c.value:age::int as child_age  
from
    json_temp d,
    lateral flatten(input => parse_json(d.json_data:children), 
                        outer => true) c
```
```
| FULL_NAME     | CHILD_IDX | CHILD_NAME | CHILD_GENDER | CHILD_AGE |
|---------------|-----------|------------|--------------|-----------|
| John Doe      | 0         | Jane       | Female       | 6         |
| John Doe      | 1         | John       | Male         | 15        |
| Mike Jones    | 0         | Earl       | Male         | 10        |
| Mike Jones    | 1         | Sam        | Male         | 6         |
| Mike Jones    | 2         | Kit        | Male         | 8         |
| Anna Karenina |           |            |              |           |
```

### Multiple Levels
Let's take this one level deeper, by analyzing the cities our persons lived in. If we take a closer look, we see that the `citiesLived` key actually contains another array made up of one dictionary per place/country, containing a `place` key denoting the city or country name and an array of the years the person lived in that place.

We can unroll both nested levels in one statement, by chaining `flatten` functions together.

Note that we are first flattening the `citiesLived` array to extract the `index` and `place` values, then flatten out the `yearsLived` array in a second pass.

Lastly, we sort the data by year for each person, to create a sort of chronology of the places they lived. 

```sql
select
    d.json_data:fullName::varchar as full_name,
    cities.index as place_idx,
    cities.value:place::varchar as place_name,
    years.value::int as year
from
    json_temp d,
    lateral flatten(input => parse_json(d.json_data:citiesLived), outer => true) cities,
    lateral flatten(cities.value:yearsLived,'') years
order by 1,4,2
```

```
| FULL_NAME     | PLACE_IDX | PLACE_NAME    | YEAR |
|---------------|-----------|---------------|------|
| Anna Karenina | 0         | Stockholm     | 1992 |
| Anna Karenina | 2         | Austin        | 1995 |
| Anna Karenina | 0         | Stockholm     | 1998 |
| Anna Karenina | 1         | Russia        | 1998 |
| Anna Karenina | 2         | Austin        | 1999 |
| Anna Karenina | 0         | Stockholm     | 2000 |
| Anna Karenina | 1         | Russia        | 2001 |
| Anna Karenina | 1         | Russia        | 2005 |
| Anna Karenina | 0         | Stockholm     | 2010 |
| John Doe      | 0         | Seattle       | 1995 |
| John Doe      | 1         | Stockholm     | 2005 |
| Mike Jones    | 3         | Austin        | 1973 |
| Mike Jones    | 0         | Los Angeles   | 1989 |
| Mike Jones    | 1         | Washington DC | 1990 |
| Mike Jones    | 0         | Los Angeles   | 1993 |
| Mike Jones    | 1         | Washington DC | 1993 |
| Mike Jones    | 2         | Portland      | 1993 |
| Mike Jones    | 0         | Los Angeles   | 1998 |
| Mike Jones    | 1         | Washington DC | 1998 |
| Mike Jones    | 2         | Portland      | 1998 |
| Mike Jones    | 3         | Austin        | 1998 |
| Mike Jones    | 3         | Austin        | 2001 |
| Mike Jones    | 0         | Los Angeles   | 2002 |
| Mike Jones    | 2         | Portland      | 2003 |
| Mike Jones    | 2         | Portland      | 2005 |
| Mike Jones    | 3         | Austin        | 2005 |
| Mike Jones    | 1         | Washington DC | 2008 |
```

## Use Case 3: Extract Values When Keys Are _Not_ Known:
For our last example, let's explore how we can extract key-value pairs from nested JSON dictionaries if we _don't know_ the keys ahead of time.
This is quite often the cases when we need to process data that contains "tags" of some sort, where a virtual grab bag of labels has been attached to a record and we don't know ahead of time how our data has been tagged.

Let's assume we didn't know which attribute was stored in our dataset for each *child*. In this case, rather than displaying the children's names, gender and age on columns, we want to show a row with the name and value of each available attribute.

To do that, we unroll the contents of the `children` dictionary and extract the keys and values into rows, like so:

```sql
select
    p.json_data:fullName::varchar as full_name,
    children.index as child_idx,
    child.key as key_name,
    child.value::varchar as key_value
from
    json_temp p,
    lateral flatten(input => parse_json(p.json_data:children), outer => true) children,
    lateral flatten(children.value, outer => true) child
```   

Again, we use the `outer` keyword to make sure we don't drop records without `children` values.

```
| FULL_NAME     | CHILD_IDX | KEY_NAME | KEY_VALUE |
|---------------|-----------|----------|-----------|
| John Doe      | 0         | age      | 6         |
| John Doe      | 0         | gender   | Female    |
| John Doe      | 0         | name     | Jane      |
| John Doe      | 1         | age      | 15        |
| John Doe      | 1         | gender   | Male      |
| John Doe      | 1         | name     | John      |
| Mike Jones    | 0         | age      | 10        |
| Mike Jones    | 0         | gender   | Male      |
| Mike Jones    | 0         | name     | Earl      |
| Mike Jones    | 1         | age      | 6         |
| Mike Jones    | 1         | gender   | Male      |
| Mike Jones    | 1         | name     | Sam       |
| Mike Jones    | 2         | age      | 8         |
| Mike Jones    | 2         | gender   | Male      |
| Mike Jones    | 2         | name     | Kit       |
| Anna Karenina |           |          |           |
```

## Summary
I hope this post provided some motivation to look to the JSON query and manipulation features in Snowflake as an alternative to preprocessing pipelines in Python and highlighted the power inherent in a distributed data warehouse platform.

For more on the topic, the [Snowflake documentation](https://docs.snowflake.net/manuals/user-guide/json-basics-tutorial.html) is a good start, or [drop us a line](/about) if you have any questions. 
