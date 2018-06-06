---
title:  "Confidence Intervals, when no real mathematicians are looking - Tableau Edition"
date:   2018-06-02 10:00AM
excerpt: "How to calculate approximations of confidence intervals for proportions & probabilities in Tableau"
categories: [tableau]
---
In our last [post](https://calogica.github.io/sql/2018/05/09/confidence-intervals-sql.html), we discussed calculating approximate confidence intervals for proportions in SQL when we don't have access to statistical distributions, like the $$beta$$ distribution. If you haven't read that one yet, I recommend you head [over there](https://calogica.github.io/sql/2018/05/09/confidence-intervals-sql.html) now to get more context on what we're trying to do.

As we saw, calculating this approximation in SQL is helpful, for example, when we need to use this confidence interval in downstream data pipelines or models.
Many times though, we just want to display the confidence interval on a dashboard in our BI tool of choice, e.g. in Tableau. In that case, it'd be better if we could dynamically calculate the CI in our BI tool.

Turns out we can apply the same technique as in the earlier post to accomplish just that. We'll use Tableau to illustrate how to do this, but most other BI tools that allow you to create custom formulas in your metric (e.g. MicroStrategy) would work here.

## The Data
We'll reuse the sample dataset we used last time, but this time we use Tableau to analyze it.

**my_order_and_exchanges_table**

| Week       | Orders | Exchanges |
|------------|-------:|----------:|
| 2018-02-25 |  10000 |      1089 |
| 2018-03-04 |  10431 |      1108 |
| 2018-03-11 |   4472 |       447 |
| 2018-03-18 |  11567 |      1172 |
| 2018-03-25 |  17864 |      1918 |
| 2018-04-01 |  16078 |      1702 |
| 2018-04-08 |  13459 |      1451 |
| 2018-04-15 |   1122 |       114 |
| 2018-04-22 |    515 |        56 |
| 2018-04-29 |  10349 |      1061 |
| 2018-05-06 |   7595 |       832 |


!["Orders and Exchanges by Week"](/assets/plots/orders_exchanges_tableau_2.png "Orders and Exchanges by Week")

We'll create a new calculated field called **Exchange Rate %** using the following formula, making sure to apply default _percent_ style formatting.
```
SUM([Exchanges])/SUM([Orders])
```
Summing the numerator and the denominator _before_ we divide the two makes sure we can correctly aggregate the metric if needed (otherwise we'd be summing the weekly exchange rates).

We note that the exchange rate % is pretty stable between 10 and 11%.

!["Exchange Rate by Week"](/assets/plots/exchange_rate_tableau.png "Exchange Rate by Week")

However, as we know from last time, we should take that with a slight grain of salt, and apply a confidence interval around this measurement to account for the varying levels of order volume from week to week.

To do that, we'll create a few more calculated fields to mirror the calculations we did in SQL in our last post.

First, we define the standard error metric **Exchange Rate % SE** (again, using the [Normal Approximation](https://en.wikipedia.org/wiki/Binomial_proportion_confidence_interval#Normal_approximation_interval){:target="_blank"}):
```
SQRT(
    [Exchange Rate %] *
    (1 - [Exchange Rate %])/SUM(Orders)
)
```

Then using 1.96 as our z-value of choice (for a [95% confidence interval](http://www.ltcconline.net/greenl/courses/201/estimation/smallConfLevelTable.htm){:target="_blank"}), we create metrics for the upper and lower bounds, like so:

**Exchange Rate % (Lower Bound)**

`[Exchange Rate %]-1.96*[Exchange Rate % SE]`

**Exchange Rate % (Upper Bound)**

`[Exchange Rate %]+1.96*[Exchange Rate % SE]`

If we're feeling fancy, we might even make the z-value a [parameter](https://onlinehelp.tableau.com/current/pro/desktop/en-us/parameters_create.html){:target="_blank"} in Tableau and use that instead, e.g.
`[Exchange Rate %]+[Z-Value]*[Exchange Rate % SE]`

Plotting all three shows us, again, that the low order volume during the week of April 22 should make us more suspicious in trusting the exchange rate for that week. If you're seeing spikes or drops in your ratio metrics, it's always helpful to look at them in the context of overall volume to guard against reacting to noise in your weekly sample.

!["Exchange Rate by Week with CI"](/assets/plots/exchange_rate_conf_int.png "Exchange Rate by Week with CI")

Next time we'll look at how we can create proper error bars using a dumbbell chart, like so:

!["Exchange Rate by Week with Error Lollipops"](/assets/plots/exchange_rate_error_bars_tableau.png "Exchange Rate by Week with Error Lollipops")
