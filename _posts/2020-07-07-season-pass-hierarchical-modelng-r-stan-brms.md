---
title: "Multi-level Modeling in RStan and brms (and the Mysteries of Log-Odds)"
date:   2020-07-05 8:00AM
excerpt: "In this post we’ll take another look at logistic regression, and in particular multi-level (or hierarchical) logistic regression in RStan brms."
categories: [r, rstan]
comments: true
---

## Overview

In this post we’ll take another look at logistic regression, and in particular multi-level (or hierarchical) logistic regression. We’ve seen Bayesian logistic regression before when we modeled [field goals in NFL football](https://calogica.com/pymc3/python/2019/12/08/nfl-4thdown-attempts.html#team-model) earlier this year, and we used multi-level models before when we looked at [Fourth-Down Attempts in NFL Football by team](https://calogica.com/pymc3/python/2019/12/08/nfl-4thdown-attempts.html#team-model). This time we’ll try to build a bit more intuition around both. Also, this will be the first post I’ll tackle in R\!

## Why R?

R has been a mainstay in statistical modeling and data science for years, but more recently has been pinned into a needless competition with Python. In fact, R has a rich and robust package ecosystem, including some of the best statistical and graphing packages out there. R, along with Python and SQL, should be part of every data scientist’s toolkit. I’ve not used R in quite a while, in favor of Python and the occasional adventure in Julia, but it’s important to recognize that we should use the right tool for the job, not just always the one that’s most convenient. Especially using the `tidyverse` package ecosystem makes data wrangling (and increasingly modeling) code almost trivial and downright fun. I encourage folks that have been away from R for a bit to give it another go\!

## Marketing Theme Park Season Passes

For this post, we’ll consider simulated sales data for a (hypothetical) theme park from chapter 9 of [“R for Marketing Research and Analytics”](http://r-marketing.r-forge.r-project.org/data.html), which inspired this post. This book really is a wide-ranging collection of statistical techniques to apply in various marketing settings and I often browse it for ideas, even if I don’t use the actual implementation.

Specifially, we’ll look at customer contacts representing attempts by the theme park to sell season passes via one of three channels - traditional mail, email and point-of-sale in the park - both as a standalone product and bundled with free parking.

The author’s have helpfully provided this data for us as a CSV with a permalink:

``` r
season_pass_data <- readr::read_csv("http://goo.gl/J8MH6A")
```

Let’s take a quick `glimpse` at the data. Looks like we have Bernoulli style data, with 3,156 records showing us whether the customer purchased a season pass (`Pass`), if they were presented with the bundle option (`Promo`) and through which `Channel` they were contacted:

``` r
glimpse(season_pass_data)
```
    ## Rows: 3,156
    ## Columns: 3
    ## $ Channel <chr> "Mail", "Mail", "Mail", "Mail", "Mail", "Mail", "Mail", "Mail…
    ## $ Promo   <chr> "Bundle", "Bundle", "Bundle", "Bundle", "Bundle", "Bundle", "…
    ## $ Pass    <chr> "YesPass", "YesPass", "YesPass", "YesPass", "YesPass", "YesPa…


All 3 columns are character columns, so we’ll want to convert them to useful factor and/or integer columns for modeling.

We’ll use `dplyr` to add a simple 1 count column `n`, and add `factor` columns for `promo` and `channel`. We’ll also convert the `Pass` variable to a Bernoulli style outcome variable of 0s and 1s.

``` r
season_pass_data <- season_pass_data %>%
    mutate(n = 1,
           bought_pass = case_when(Pass == "YesPass" ~ 1, TRUE ~ 0),
           promo = factor(Promo, levels = c("NoBundle", "Bundle")),
           channel = factor(Channel, levels = c("Mail", "Park", "Email"))
           )
```

When creating factor variables it’s usually a good idea to confirm the factor ordering to make sure it aligns with our expectations, which we can do with the `contrasts` function:

``` r
contrasts(season_pass_data$promo)
```

    ##          Bundle
    ## NoBundle      0
    ## Bundle        1

``` r
contrasts(season_pass_data$channel)
```

    ##       Park Email
    ## Mail     0     0
    ## Park     1     0
    ## Email    0     1

Next up, let’s convert our Bernoulli style data to Binomial data, by grouping and summarizing, to make our models run more efficiently.

``` r
season_pass_data_grp <- season_pass_data %>% 
    group_by(promo, channel) %>%
    summarise(bought_pass = sum(bought_pass), 
              n = sum(n)) %>%
    ungroup() 

season_pass_data_grp
```

    ## # A tibble: 6 x 4
    ##   promo    channel bought_pass     n
    ##   <fct>    <fct>         <dbl> <dbl>
    ## 1 NoBundle Mail            359   637
    ## 2 NoBundle Park            284   333
    ## 3 NoBundle Email            27   512
    ## 4 Bundle   Mail            242   691
    ## 5 Bundle   Park            639   862
    ## 6 Bundle   Email            38   121

## Exploring the Data

Next, let’s use `dplyr` and `ggplot2` to look at a few different cuts of this data to get a sense of how we can answer some of the business questions we might encounter.

For example, we might get asked:

### 1\) How many customers bought a season pass by channel, in a bundle or no bundle?

``` r
season_pass_data_grp %>%
    select(channel, promo, bought_pass) %>%
    pivot_wider(names_from = promo, values_from = bought_pass) %>%
    adorn_totals("col") 
```
    ##  channel NoBundle Bundle Total
    ##     Mail      359    242   601
    ##     Park      284    639   923
    ##    Email       27     38    65

(Note: we use the extra-handy `adorn_totals` function from the `janitor` package here)

Or visually:

``` r
season_pass_data_grp %>% 
    ggplot(aes(x = channel, y = bought_pass, group = promo, fill = promo)) +
    geom_col() + 
    scale_y_continuous() +
    scale_fill_ft() +
    labs(x = "",
       y = "# Bought Season Pass",
       title = "Customers by Channel",
       subtitle = "by Promotion (Bundle/NoBundle)"
       )
```

<img src="/assets/plots/r-mlm-season-pass/season_pass_plot_1-1.png" style="display: block; margin: auto auto auto 0;" />

We note that `Park` is our biggest sales channel, while `Email` had by far the lowest overall sales volume.

### 2\) What percentage of customers bought a season pass by channel, in a bundle or no bundle?

``` r
season_pass_data_grp %>% 
    group_by(channel) %>%
    summarise(bought_pass = sum(bought_pass), 
              n = sum(n),
              percent_bought = bought_pass/n) %>%
    ggplot(aes(x = channel, 
               y = percent_bought, 
               fill = channel, 
               label = scales::percent(percent_bought))) + 
    geom_col(width = .5) + 
    coord_flip() +
    theme(legend.position = "none") +
    geom_text(hjust = "outward", nudge_y=.01, color="Black") + 
    scale_fill_ft() +
    scale_y_continuous(labels = NULL) +
    labs(x = "",
       y = "% Bought Season Pass by Channel",
       title = "% of Customers by Channel"
       )
```

<img src="/assets/plots/r-mlm-season-pass/season_pass_plot_2-1.png" style="display: block; margin: auto auto auto 0;" />

`Email` seems to also have the lowest take rate of all channels, with only 10% of contacted customer buying a season pass. At the same time, the high take rate (77%) of customers in the park could be indication of selection basis, wherein customers already in the park have demonstrated a higher propensity to purchase theme park passes.

### 3\) What percentage of customers that bought a season pass bought it in a bundle by channel?

``` r
season_pass_data_grp %>%
    select(channel, promo, bought_pass) %>%
    pivot_wider(names_from = promo, values_from = bought_pass) %>%
    mutate(percent_bundle = Bundle/(NoBundle + Bundle)) -> season_pass_data_grp_pct_bundle

season_pass_data_grp_pct_bundle
```

    ## # A tibble: 3 x 4
    ##   channel NoBundle Bundle percent_bundle
    ##   <fct>      <dbl>  <dbl>          <dbl>
    ## 1 Mail         359    242          0.403
    ## 2 Park         284    639          0.692
    ## 3 Email         27     38          0.585

``` r
season_pass_data_grp_pct_bundle %>% 
    ggplot(aes(x = channel, 
               y = percent_bundle, 
               fill = channel, 
               label = scales::percent(percent_bundle)
               )
           ) +
    geom_col(width = .5) + 
    coord_flip() +
    theme(legend.position = "none") +
    geom_text(hjust = "outward", nudge_y=.01, color="Black") + 
    scale_y_continuous(labels = NULL) +
    scale_fill_ft() +
    labs(x = "",
       y = "% Bought Season Pass w/Bundle",
       title = "% of Bundle Customers by Channel"
       )
```

<img src="/assets/plots/r-mlm-season-pass/season_pass_plot_3-1.png" style="display: block; margin: auto auto auto 0;" />

Again, customers in the park have the highest percentage of season passes sold in the bundle. We could argue that since they’re already showing higher motivation to buy a season pass, the upsell to pass bundled with parking is comparatively easier.

Interestingly, almost 60% of customers contacted via email that purchased a season pass bought it as part of the bundle.

Given the relatively small number of overall email-attributed sales, it makes sense to investigate further here to see if bundling is in fact a valuable sales strategy for digital channels vs mail and in the park.

## A Baseline Model

In classical modeling, our first instinct here would be to model this as logistic regression, with `bought_pass` as our response variable. So, if we wanted to measure the overall effectiveness of our bundle offer, we’d set up a simple model using the `glm` module and get a `summary` of estimated coefficients. However, as good Bayesians that value interpretable uncertainty intervals, we’ll go ahead and use the excellent `brms` library that makes sampling via RStan quite easy.

Instead of relying on the default priors in `brms`, we’ll use a fairly diffuse `Normal` prior for intercept and slope.

``` r
base_line_promo_model <- brm(bought_pass | trials(n) ~ 1 + promo,
                             prior = c(prior(normal(0, 5), class = Intercept),
                                       prior(normal(0, 5), class = b)),
                             data = season_pass_data_grp,
                             family = binomial(link = "logit"),
                             iter = iter
                             )
```

We’ll take a quick look at chain divergence, mostly to introduce the excellent `mcmc` plotting functions from the `bayesplot` package.

``` r
mcmc_trace(base_line_promo_model, regex_pars = c("b_"), facet_args = list(nrow = 2))
```

<img src="/assets/plots/r-mlm-season-pass/plot_base_line_promo_model_trace-1.png" style="display: block; margin: auto auto auto 0;" />

We note that our chains show convergence and are well-mixed, so we move on to taking a look at the estimates:

``` r
summary(base_line_promo_model)
```

    ##  Family: binomial 
    ##   Links: mu = logit 
    ## Formula: bought_pass | trials(n) ~ 1 + promo 
    ##    Data: season_pass_data_grp (Number of observations: 6) 
    ## Samples: 4 chains, each with iter = 10000; warmup = 5000; thin = 1;
    ##          total post-warmup samples = 20000
    ## 
    ## Population-Level Effects: 
    ##             Estimate Est.Error l-95% CI u-95% CI Rhat Bulk_ESS Tail_ESS
    ## Intercept      -0.19      0.05    -0.29    -0.09 1.00    16450    12347
    ## promoBundle     0.39      0.07     0.25     0.53 1.00    17456    12744
    ## 
    ## Samples were drawn using sampling(NUTS). For each parameter, Bulk_ESS
    ## and Tail_ESS are effective sample size measures, and Rhat is the potential
    ## scale reduction factor on split chains (at convergence, Rhat = 1).

``` r
mcmc_areas(
    base_line_promo_model,
    regex_pars = "b_",
    prob = 0.95, 
    point_est = "median",
    area_method = "equal height"
    ) +
    geom_vline(xintercept = 0, color = "red", alpha = 0.6, lwd = .8, linetype = "dashed") +
    labs(
        title = "Effect of Bundle Promotion on Sales"
    )
```

<img src="/assets/plots/r-mlm-season-pass/plot_base_line_promo_model_areas-1.png" style="display: block; margin: auto auto auto 0;" />

The slope coefficient `promoBundle` is positive and significant (does not contain 0 in the uncertainty interval). The value of `0.39` represents the effect of the `Bundle` treatment in terms of log-odds, i.e. bundling increases the log odds of buying a season pass by `0.39`. We can convert that to a % by exponentiating the coefficients (which we get via `fixef`) to get the % increase of the odds:

``` r
exp(fixef(base_line_promo_model))
```

    ##              Estimate Est.Error      Q2.5     Q97.5
    ## Intercept   0.8254383  1.053558 0.7448517 0.9146389
    ## promoBundle 1.4751278  1.075495 1.2781352 1.7024751

In terms of percent change, we can say that the odds of a customer buying a season pass when offered the bundle are 47.5% higher than if they’re not offered the bundle.

### Aside: what the heck are log-odds anyway?

Log-odds, as the name implies are the logged odds of an outcome. For example, an outcome with odds of `4:1`, i.e. a probability of 80% (`4/(4+1)`) has log-odds of `log(4/1) = 1.386294`.

Probability, at its core is just counting. Taking a look at simple crosstab of our observed data, let’s see if we can map those log-odds coefficients back to observed counts.

``` r
season_pass_data %>% 
    group_by(promo) %>%
    summarise(bought_pass = sum(bought_pass),
              did_not_buy = sum(n) - sum(bought_pass)) %>%
    adorn_totals(c("row", "col"), name="total") %>%
    mutate(percent_bought = bought_pass/total)
```

    ##     promo bought_pass did_not_buy total percent_bought
    ##  NoBundle         670         812  1482      0.4520918
    ##    Bundle         919         755  1674      0.5489845
    ##     total        1589        1567  3156      0.5034854

We estimated an intercept of `-0.19`, which are the log-odds for `NoBundle` (the baseline). We observed `670` of `1,482` customers that were **not** offered the bundle bought a season pass vs `812` that didn’t buy. With odds defined as *bought/didn’t buy*, the `log` of the NoBundle buy odds is:

``` r
odds_no_bundle <- 670/812
log(odds_no_bundle)
```

    ## [1] -0.1922226


While our estimated slope of `0.39` for `Bundle` is the `log` of the **ratio** of *buy/didn’t buy* odds for Bundle vs NoBundle:

``` r
odds_no_bundle <- 670/812
odds_bundle <- 919/755
log(odds_bundle/odds_no_bundle)
```

    ## [1] 0.388791


If we do this without taking any logs,

``` r
odds_no_bundle
```

    ## [1] 0.8251232


``` r
odds_bundle/odds_no_bundle
```
    
    ## [1] 1.475196


we see how this maps back to the exponentiated slope coefficient from the model above:

``` r
exp(fixef(base_line_promo_model))
```

    ##              Estimate Est.Error      Q2.5     Q97.5
    ## Intercept   0.8254383  1.053558 0.7448517 0.9146389
    ## promoBundle 1.4751278  1.075495 1.2781352 1.7024751

We can think of `1.4750` as the **odds ratio** of Bundle vs NoBundle, where ratio of `1` would indicate no improvement.

What’s more, we can link the overall observed % of sales by Bundle vs Bundle to the combination of the coefficients. For predictive purposes, logistic regression in this example would compute the log-odds for a case of `NoBundle (0)` roughly as:

``` r
plogis(-0.19 + 0.39*0) 
```

    ## [1] 0.4526424


And `Bundle (1)` as :

``` r
plogis(-0.19 + 0.39*1) 
```

    ## [1] 0.549834


Which maps back to our observed proportions of 45% and 55% in our counts above.

We can also show this via the `predict` function for either case:

``` r
newdata <- data.frame(promo = factor(c("NoBundle", "Bundle")), n = 1)

predict(base_line_promo_model, newdata)[c(1:2)]
```

    ## [1] 0.45105 0.54905

Logistic regression is probably one of the most underrated topics in modern data science.

(Thanks to the folks at he UCLA Stats department for [this detailed writeup](https://stats.idre.ucla.edu/other/mult-pkg/faq/general/faq-how-do-i-interpret-odds-ratios-in-logistic-regression/) on this topic.)

## Is this right model?

Back to our model\!

However, this simple model fails to take `Channel` into consideration and is not actionable from a practical marketing standpoint where channel mix is an ever-present optimization challenge. In other words, while the model itself is fine and appears to be a good fit, it’s not really an appropriate “small world” model for our “large world”, to quote [Richard McElreath](https://books.google.com/books?id=T3FQDwAAQBAJ&pg=PA19&lpg=PA19&dq=mcelreath+small+world+big+world&source=bl&ots=vsrrBaL97W&sig=ACfU3U1qDTwHgFTyEPBmxAhkQmdX5FgC0Q&hl=en&sa=X&ved=2ahUKEwih9c6btbbqAhUVJzQIHXRlD2IQ6AEwC3oECAoQAQ#v=onepage&q=mcelreath%20small%20world%20big%20world&f=false).

Simply modeling `Channel` as another independent (dummy) variable would also likely misrepresent the actual data generating process, since we know from our EDA above that `Channel` and `Promo` seem to depend on one another.

So, let’s try to model this dependency with another common technique in classical modeling, *interaction terms*.

## Modeling Interactions

Again using the `brms` library, it’s easy to add interaction terms using the `*` formula convention familiar from `lm` and `glm`. This will create both individual slopes for each variable, as well as the interaction term:

``` r
promo_channel_model_interactions <- brm(bought_pass | trials(n) ~ promo*channel, 
                                        prior = c(prior(normal(0, 5), class = Intercept),
                                                  prior(normal(0, 5), class = b)),
                                        data = season_pass_data_grp,
                                        family = binomial(link = "logit"),
                                        iter = iter)
```

Again, our model converged well and we observe well-mixed chains in the traceplot:

``` r
mcmc_trace(promo_channel_model_interactions, regex_pars = c("b_"), facet_args = list(nrow = 3))
```

<img src="/assets/plots/r-mlm-season-pass/plot_promo_channel_model_interactions_trace-1.png" style="display: block; margin: auto auto auto 0;" />

(We’ll forgo convergence checks from here on out for this post, but it’s never a bad idea to inspect your chains for proper mixing and convergence.)

``` r
summary(promo_channel_model_interactions)
```

    ##  Family: binomial 
    ##   Links: mu = logit 
    ## Formula: bought_pass | trials(n) ~ promo * channel 
    ##    Data: season_pass_data_grp (Number of observations: 6) 
    ## Samples: 4 chains, each with iter = 10000; warmup = 5000; thin = 1;
    ##          total post-warmup samples = 20000
    ## 
    ## Population-Level Effects: 
    ##                          Estimate Est.Error l-95% CI u-95% CI Rhat Bulk_ESS
    ## Intercept                    0.25      0.08     0.10     0.41 1.00    16692
    ## promoBundle                 -0.87      0.11    -1.09    -0.65 1.00    13784
    ## channelPark                  1.51      0.17     1.18     1.86 1.00    11049
    ## channelEmail                -3.15      0.21    -3.58    -2.75 1.00    12707
    ## promoBundle:channelPark      0.16      0.21    -0.25     0.56 1.00    10857
    ## promoBundle:channelEmail     2.98      0.30     2.39     3.57 1.00    13243
    ##                          Tail_ESS
    ## Intercept                   14853
    ## promoBundle                 13391
    ## channelPark                 11780
    ## channelEmail                12117
    ## promoBundle:channelPark     12026
    ## promoBundle:channelEmail    12156
    ## 
    ## Samples were drawn using sampling(NUTS). For each parameter, Bulk_ESS
    ## and Tail_ESS are effective sample size measures, and Rhat is the potential
    ## scale reduction factor on split chains (at convergence, Rhat = 1).
    

``` r
mcmc_areas(
    promo_channel_model_interactions,
    regex_pars = "b_",
    prob = 0.95, # 80% intervals
    prob_outer = 1, # 99%
    point_est = "median",
    area_method = "equal height"
    ) +
    geom_vline(xintercept = 0, color = "red", alpha = 0.6, lwd = .8, linetype = "dashed") +
    labs(
        title = "Effect of Channel and Bundle Promotion",
        subtitle = "with interactions"
    )
```

<img src="/assets/plots/r-mlm-season-pass/plot_promo_channel_model_interactions_areas-1.png" style="display: block; margin: auto auto auto 0;" />

Three things immediately come to our attention:

- the `Email` channel is associated with a -3.15 decrease in log odds of selling a season pass (vs the baseline channel `Mail` )
- however, the interaction term `promoBundle:channelEmail`, i.e. the effect of the `Bundle` promo given the `Email` channel shows a \~3x increase in log-odds over the baseline
- interestingly, the `Park` channel does not seem to significantly benefit from offering a bundle promotion, shown by it’s small coefficient with uncertainty interval spanning `0`

So, while `Email` itself has shown to be the least effective sales channel, we see that offering a bundle promotion in emails seems to make the most sense. Perhaps, customers on our email list are more discount motivated than customers in other channels.

At the same time, our customers in the park, as we’ve speculated earlier, seem to have higher price elasticity than mail or email customers, making the park a better point-of-sale for non-bundled (and presumably non-discounted) SKUs.

In “R for Marketing Research and Analytics”, the authors also point out that the interaction between `channel` and `promo` in this data points to a case of [Simpson’s Paradox](https://en.wikipedia.org/wiki/Simpson%27s_paradox) where the aggregate effect of `promo` is different (potentially and misleading), compared to the effect at the channel level.

## Multi-Level Modeling

Interaction terms, however useful, do not fully take advantage of the power of Bayesian modeling. We know from our EDA that email represent a small fraction of our sales. So, when computing the effects of `Email` and `Promo` on `Email`, we don’t fully account for inherent lack of certainty as a result of the difference in sample sizes between channels. A more robust way to model interactios of variables in Bayesian model are *multilevel* models. They offer both the ability to model interactions (and deal with the dreaded collinearity of model parameters) and a built-in way to regularize our coefficient to minimize the impact of outliers and, thus, prevent overfitting.

In our case, it would make the most sense to model this with both varying intercepts and slopes, since we observed that the different channels appear to have overall lower baselines (arguing for varying intercepts) and also show different effects of offering the bundle promotion (arguing for varying slopes). In other cases though, we may need to experiment with different combinations of fixed and varying parameters.

Luckily, it’s a fairly low-code effort to add grouping levels to our model. We will model both a varying intercept (`1`) and varying slope (`promo`) by `channel`, removing the standard population level intercept (`0`) and slope. (The `||` tells `brms` not to bother to compute correlations.)

``` r
promo_channel_model_hierarchical <- brm(bought_pass | trials(n) ~ 0 + (1 + promo || channel),
                                        prior = c(prior(normal(0, 5), class = sd)),
                                        data = season_pass_data_grp,
                                        control = list(adapt_delta = 0.95),
                                        family = binomial(link = "logit"),
                                        iter = iter
                                        )
```

This time we’ll use the `broom` package to `tidy` up the outputs of our model so that we can inspect the varying parameters of our model more easily:

``` r
tidy(promo_channel_model_hierarchical, prob = 0.95)
```

    ##                           term    estimate  std.error        lower       upper
    ## 1        sd_channel__Intercept   2.8720920 1.48675939   1.19672735   6.8935430
    ## 2      sd_channel__promoBundle   2.1567263 1.27624244   0.81757904   5.6469032
    ## 3    r_channel[Mail,Intercept]   0.2537450 0.07963305   0.09733499   0.4116723
    ## 4    r_channel[Park,Intercept]   1.7493075 0.15321905   1.45558853   2.0554104
    ## 5   r_channel[Email,Intercept]  -2.8482983 0.19604272  -3.25390531  -2.4838843
    ## 6  r_channel[Mail,promoBundle]  -0.8698846 0.11171531  -1.09155716  -0.6521658
    ## 7  r_channel[Park,promoBundle]  -0.6940342 0.17151105  -1.03623924  -0.3635511
    ## 8 r_channel[Email,promoBundle]   2.0272838 0.28068435   1.48506859   2.5846931
    ## 9                         lp__ -31.5439369 2.54313133 -37.41339907 -27.5445567

Another benefit of multi-level models is that each level is explicitly modeled, unlike traditional models where we typically model n-1 coefficients and are always left to interpret coefficients against some un-modeled baseline.

From the output above, we can see that `Email` in general is still performing worse vs the other channels judging from its low negative coefficient, while the effect of the `Bundle` promo for the `Email` channel is significantly positive at \~2 increase in log-odds. However, compared to our single-level interaction models, we see that the hierarchical did a better job constraining the estimate of the effect of offering the bundle in emails by shrinking the estimate a bit towards the group mean.

Visualizing this as a ridge plot, it’s more clear how the `Bundle` effect for `Email` is less certain than for other models, which makes intuitive sense since we have a lot fewer example of email sales to draw on. However, it appears to be the only channel where bundling free parking makes a real difference in season pass sales.

``` r
mcmc_areas(
    promo_channel_model_hierarchical,
    regex_pars = "r_",
    prob = 0.8, # 80% intervals
    point_est = "median",
    area_method = "equal height"
    ) +
    geom_vline(xintercept = 0, color = "red", alpha = 0.6, lwd = .8, linetype = "dashed") +
    labs(
        title = "Effect of Channel and Bundle Promotion",
        subtitle = "hierarchical model: random intercept and slope"
    )
```

<img src="/assets/plots/r-mlm-season-pass/plot_promo_channel_model_hierarchical_areas-1.png" style="display: block; margin: auto auto auto 0;" />

So, while we’ve seen that email response and take rates are the lowest of all channels, we can confidently tell our marketing partners that offering bundling via email has a positive effect that is worth studying more and gathering more data. Since email tends to be a cheaper alternative to conventional in-home mails, and certainly cheaper than shuttling people into the park, the lower response rate needs to be weighed against channel cost.

## Summary

Let’s wrap up with a few take aways:

- Although it might have been obvious in this example dataset, but a first step in modeling is to make sure our model captures the true data generating process adequately, so we can ultimately answer the most meaningful business questions with confidence. So, even a well fitting model may be the wrong model in a given context. Or in short, make sure “small world” represents “large world” appropriately.
- Interaction terms are an easy first step to add dependencies to our model. However, for larger models with many coefficients, they can become difficult to interpret and don’t easily allow for regularization of parameters.
- From a modeling perspective, multi-level or hierarchical models are a very flexible way to approach regression models. They allow us to encode relationships that help create stronger estimates by pooling (sharing) data across grouping levels, while also helping to regularize estimates to avoid overfitting.
- Through libraries like `brms`, implementing hierarchical models in R becomes only somewhat more involved than classical regression models coded in `lm` or `glm`.

So, for anything but the most trivial examples, Bayesian hierarchical models should really be our default choice.

## Resources

I’ve found these links helpful whenever I’ve worked on multi-level Bayesian models and/or R:

- [Richard McElreath’s book, Statistical Rethinking](https://xcelab.net/rm/statistical-rethinking/), including his invaluable [lectures on video](https://github.com/rmcelreath/statrethinking_winter2019)
- [Solomon Kurz’ adaptation](https://bookdown.org/connect/#/apps/4857/access) of Statisticial Rethinking to the tidyverse
- [The R Graph Gallery](https://www.r-graph-gallery.com/index.html)
- [HBR Themes for ggplot](https://github.com/hrbrmstr/hrbrthemes)
- [The `brms` package documentation](https://www.rdocumentation.org/packages/brms/versions/2.13.0)
- [The Stan User Guide](https://mc-stan.org/users/documentation/)
- [“FAQ: HOW DO I INTERPRET ODDS RATIOS IN LOGISTIC REGRESSION?”](https://stats.idre.ucla.edu/other/mult-pkg/faq/general/faq-how-do-i-interpret-odds-ratios-in-logistic-regression/)
- [“R for Marketing Research and Analytics”](http://r-marketing.r-forge.r-project.org/data.html)
