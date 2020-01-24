---
title:  "Updated: How to Backup Snowflake Data - GCS Edition" 
date:   2020-01-23 7:00PM
excerpt: "Updated Post: How to backup a Snowflake database to S3 or GCS, contributed by Taylor Murphy"
categories: [sql, snowflake, dbt]
comments: true
---

{: .notice--info}
Did you know you can now [sign up](/signup) for weekly-ish updates to our blog via email?

Many thanks to [Taylor Murphy at Gitlab](https://gitlab.com/tayloramurphy) who submitted the first ever [PR](http://github.com/calogica/calogica.com/pull/1) to the blog!

Taylor added  super valuable steps to the ["How to Backup Snowflake Data"](/sql/snowflake/dbt/2019/09/09/snowflake-backup-s3.html) post from September that show how to use the same approach if you're running Snowflake on **Google Cloud**.

Gitlab is using this approach as part of their dbt pipeline within Airflow. (See Taylor's MR over at [Gitlab](https://gitlab.com/gitlab-data/analytics/merge_requests/2262).)

Snowflake is now generally available on GCP, so I'm sure this will be very helpful for anyone looking to use dbt to backup their Snowflake tables to GCS.

If you have any suggestions or corrections to any of the posts on this blog, feel free to [submit a PR](http://github.com/calogica/calogica.com/) via Github!