---
title:  "Modeling Conversion Cohorts using Survival Models with the convoys package"
date:   2019-09-16 8:00AM
excerpt: "How to model marketing conversion cohorts using survival models with the convoys package."
categories: [data-science, python]
---

{: .notice--info}
In this post, we take a look at how to model conversion cohorts using parametric and non-parametric survival models with the convoys package.

## Censured Data
One of the challenges with modeling, and therefore trying to predict conversion data (for example, in a SaaS or eCommerce setting) is that we don't have a full picture of who has yet to convert, or will never convert. In the statistical literature, this is known as _censured_ data, more specifically in the case of conversion data, _right censored_ data since the "right" side of the conversion timeline is not fully known to us.

## Survival Models
Thus, a common approach to modeling conversion data is to use survival models that take censored data into account. 
 (Or maybe not _that_ common, see ["Conversion rates – you are (most likely) computing them wrong"](https://erikbern.com/2017/05/23/conversion-rates-you-are-most-likely-computing-them-wrong.html).)

### Non-Parametric

### Parametric

### Caveats
This approach comes with another caveat though: in conversion scenarios, many cases (i.e. users) will _never_ have an event, i.e. conversion. This is in contrast to classical survival models, where we assume that most/all cases will _eventually_ have an event, we just don't know exactly when.

## Convoys
In response, Erik Bernhardsson has created the very helpful `convoys` package for Python that allows us to better model survival data with sparse survival rates.