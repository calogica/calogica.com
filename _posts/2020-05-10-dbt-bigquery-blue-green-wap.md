---
title:  "Blue-Green Data Warehouse Deployments (Write-Audit-Publish) with BigQuery and dbt" 
date:   2020-05-24 8:00AM
excerpt: "How to implement the Write-Audit-Publish (WAP) pattern using dbt on BigQuery"
categories: [sql, bigquery, dbt]
comments: true
---

On a current project with the data team at [Quibi](https://quibi.com), we were confronted with the challenge of how to incorporate the agility of developing dbt models against our BigQuery data warehouse with a process that gave us peace of mind that the data exposed via our BI tools is audited and accurate before publishing it to data consumers.

I had a lot of fun talking to the dbt Community about how to implement Blue-Green deployments and the WAP pattern for BigQuery data warehouse projects with **dbt** at the last dbt Office Hours. I put together a few slides for the talk, available [here](assets/wap_dbt_bigquery.pdf). By now, the folks at Fishtown may have already posted the video, so if you're interested, check it out by joining the [Community](https://community.getdbt.com/)!