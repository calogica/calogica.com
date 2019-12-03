---
title:  "What We're Reading - Week of 11/25/2019"
date:   2019-11-26 8:00AM
excerpt: "What We're Reading - Week of 11/25/2019: articles and posts we enjoyed "
categories: [reads]
---

## [Predicting Time to Cook, Arrive, and Deliver at Uber Eats](https://www.infoq.com/articles/uber-eats-time-predictions/?utm_source=email&utm_medium=ai-ml-data-eng&utm_campaign=newsletter&utm_content=11262019)
Interesting walk-through of Uber Eats' time to delivery prediction models, built on top of its ML platform Michelangelo.

I particularly enjoyed this paragraph, highlighting how a lack of process and domain knowledge will come to bite even the smartest engineers:

> Apparently, it’s not scientific to use 25 mins for all orders from that restaurant regardless whether eaters order 1 dish or 10 dishes - the same goes for  the 5 min setting for all delivery partners’ travel time. As a result, it caused lots of confusion among all the partners, leading to problems such as the inability for eaters to track their food accurately, delivery partners extended waiting period at restaurants or restaurants having no idea where the delivery partner was while the food got cold.

![Uber Eats Dispatch](https://res.infoq.com/articles/uber-eats-time-predictions/en/resources/1dispatch-system-dispatch-timing-1573831753582.jpg)

## [No, Bayes does not like Mayor Pete](https://statmodeling.stat.columbia.edu/2019/11/23/pitfalls-of-using-implied-betting-market-odds-to-estimate-electability/)
Andrew Gelman with an interesting example of Bayesian workflow in practice and the pitfalls of using implied betting market odds to estimate electability. 

Predictions are hard, especially about the future.

## [Anomaly Detection Toolkit (ADTK)](https://arundo-adtk.readthedocs-hosted.com/en/latest/index.html)
This new library looks like a very robust way to do timeseries outlier and anomaly detection in Python. We're looking forward to giving this a try on a project soon.
> Anomaly Detection Toolkit (ADTK) is a Python package for unsupervised / rule-based time series anomaly detection.

> This package offers a set of common detectors, transformers and aggregators with unified APIs, as well as pipe classes that connect them together into a model. It also provides some functions to process and visualize time series and anomaly events.

## [No More Gurus](https://www.collaborativefund.com/blog/no-more-gurus/)
Anything [Morgan Housel](https://twitter.com/morganhousel) writes is worth reading.
> There is a difference between an expert, whose talent should always be celebrated, and a guru, whose bad ideas should never be questioned.

## [Does LA Actually Have More Vacant Units Than Homeless People? Our Mea Culpa](https://laist.com/2019/11/25/does_la_actually_have_more_vacant_units_than_homeless_people_our_mea_culpa.php?te=1&nl=california-today&emc=edit_ca_20191126?campaign_id=49&instance_id=14118&segment_id=19108&user_id=d75da2816750ee89699a8d7bf1c10464&regi_id=7774356420191126)
A rare retraction of a "data-driven" article on homelessness shows that proper data analysis is hard and requires rigorous methodology and solid domain knowledge if you want to make any plausible claims.

The post also includes some astounding numbers:
> That figure — 70,545 homeless children — is alarming. Few of these children physically live on the street, but many of them do live in vehicles — typically considered the most common (and most undercounted) form of homelessness in Los Angeles. 