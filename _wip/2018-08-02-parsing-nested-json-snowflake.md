---
title:  "Parsing Nested JSON Dictionaries in SQL - Snowflake Edition"
date:   2018-08-03 8:00AM
excerpt: "How to parse nested dictionaries in Snowflake table columns using SQL"
categories: [sql]
comments: true
---
Over the last couple of months working with clients, I've been mostly working in SQL and mostly with Snowflake. Working with a new dataset containing nested JSON in table columns, gave me chance to test drive the powerful, if under-documented JSON features of Snowflake.
For this project, I hoped to avoid a more complex Python-based processing pipeline, so I figured I'd see how far we can take JSON parsing in SQL.

We'll explore 3 examples:
- How to simple extract values from single-level JSON if we know the name of the keys ahead of time (sort of as a warm up)
- How to extract values from known JSON keys that are nested one or more levels deep (fun!)
- How to extract the keys _and_ values from JSON dictionaries, even nested ones

For the examples below, we'll assume the following data.

```json
{
    "geo":
    { "countries":
        {
            "us": 1.23,
            "uk": 2.34,
            "jp": 8.98
        },
        "states":
        {
            
        }
    }
}
```
