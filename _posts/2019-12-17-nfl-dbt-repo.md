---
title:  "Introducing the nfl-dbt repo"
date:   2019-12-16 10:00AM
excerpt: "Introducing the nfl-dbt repo: analytical models for dbt play-by-play data"
categories: [dbt]
---

{: .notice--info}
Did you know you can now [sign up](/signup) for weekly-ish updates to our blog via email?

In my last post, ["Bayesian Modeling of NFL Football Fourth Down Attempts with PyMC3"](/pymc3/python/2019/12/08/nfl-4thdown-attempts.html), I referenced the `nflscrapR-data` repo where I had sourced the relevant play-by-play for NFL games from 2009 through 2019.
After initially loading this data to BigQuery and querying the raw tables, I quickly realized that this calls for a clean and repeatable data loading and transformation process if this dataset is to be used for model building or teaching purposes.

So, I'm happy to release v1 of the [nfl-dbt repo](https://github.com/clausherther/nfl-dbt)!

This repo contains [dbt](https://www.getdbt.com) models to transform NFL Play-by-Play (pbp) data sourced from [nflscrapR-data](https://github.com/ryurko/nflscrapR-data.git) into analytical models.

The repo currently assumes that raw data is loaded to and transformed on a local or remote **PostgreSQL** instance. The load script outlined below and the dbt models could be easily modified to work with other databases supported by dbt such as Snowflake, BigQuery or Redshift. [PRs welcome!](https://github.com/clausherther/nfl-dbt/issues)

## Update Frequency
The `nflscrapR-data` repo is updated with some regularity, but since this is a voluntary and free resource (thanks to [Ron Yurko](https://twitter.com/Stat_Ron)!), we can't rely on play data being updated weekly. So, this dataset and the analytical models are best used for teaching and model building purposes, and perhaps less so for weekly decision on sports bets etc.

## Models
- `dates`: list of all game dates by season and season type (`PRE`, `REG`, `POST`)
- `games`: game id, dates, teams and final scores by game 
- `players`: player id and name for every player
- `plays`: combines play data from all available seasons (2009 to 2019) into a single table for easier analysis
- `teams`: team code and consolidated code, in case of team moves of renames
- `teams_players`: team rosters by season, showing player and (primary) position for the season

### Notes 
- a few missing `player_id` values in the `players` and `teams_player` models have been (at least attempted to be) fixed
- any duplicate plays (likely a result of the scraping process) are removed from `plays`

## Data Load
The repo assumes that the raw scraped data has been loaded to a **PostgreSQL** database, with one raw file corresponding to a single table in a database called `raw`.

The included Python script [`prep.py`](https://github.com/clausherther/nfl-dbt/blob/master/prep.py) is intended to do the following:
- Clone and/or locally refresh the `nflscrapR-data` repo
- Create empty tables in a local Postgres instance
- Load raw data files to Postgres using a `dbt run-operation` to load each file using Postgres' `copy` command

The `prep.py` file can be easily configured to work with a remote Postgres server, e.g. hosted on AWS RDS. With a little bit of [extra work](https://github.com/clausherther/nfl-dbt/issues) this can also be modified to work with Snowflake, BigQuery or Redshift.

## Future Work
The following items would make great natural extensions and improvements to the repo:
- Update `prep.py` to use connection info from `~/.dbt/profiles.yml`
- Add support for Snowflake, BigQuery and Redshift
- Add report models to more easily enable analytical models:
    - Player stats
    - Game stats
    - Season stats
- Remove dependency on `nflscrapR-data` and include `R` scripts to scrape the data independently

As always, let me know what you think and I'm looking forward to hearing what you do with the data!