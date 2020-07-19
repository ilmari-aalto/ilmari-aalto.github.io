---
layout: post
title:  "Workaround for full outer joins"
date:   2020-07-19
categories: sql
---
Full outer joins are often confusing, hard to write correctly and easy to break when extending the existing code. I use perhaps one full outer join every two months or so, but most of the time try to work around them. In this post I present `union all` as a possible workaround for a full outer join.

## Full outer join example

Let imagine we try to join together impressions, clicks and conversions by `campaign_id`. The first thought would be to left join them together starting with impressions. All three events are independent, however, so we need full joins. The query could look for example like this:

{% highlight sql %}
select
  campaign_id,
  count(impressions.campaign_id) as impressions,
  count(clicks.campaign_id) as clicks,
  count(conversions.campaign_id) as conversions
from impressions
full outer join clicks using (campaign_id)
full outer join conversions using (campaign_id)
group by 1;
{% endhighlight %}

This is fine and very clear so far, mainly thanks to the `using` functionality that does the magic for us on the `campaign_id` column. But let's extend the example a bit and add the event date `dt`. Since this column is not part of the join with `using` it could be `null` for any of the source tables. So we'd have to `coalesce` the column like this:

{% highlight sql %}
select
  campaign_id,
  coalesce(impressions.event_ts,
    clicks.event_ts,
    conversions.event_ts) :: date as dt,
  count(impressions.campaign_id) as impressions,
  count(clicks.campaign_id) as clicks,
  count(conversions.campaign_id) as conversions
from impressions
full outer join clicks using (campaign_id)
full outer join conversions using (campaign_id)
where dt between '2020-07-01' and '2020-07-19'
group by 1, 2;
{% endhighlight %}

How about a filter on a column that's only in one of the tables, such as `amount` in `conversions`? If we simply write `where amount > 0` then the full outer join would switch to an inner join, and that would not be the desired result. So instead we could write `where (amount > 0 or amount is null)` but that's hacky. Besides, what happens if `amount` is nullable in `conversions`? Sure, we can get around it by preparing the conversion data in a subquery, for example:

{% highlight sql %}
with conversions_with_amount as (
  select
    campaign_id,
    event_ts
  from conversions
  where amount > 0
)
...
{% endhighlight %}

But to be honest, that's already quite complicated for the problem at hand.

## Mimic full outer join with union all

Instead, we could switch to `union all`'s and use ones together with zeros or nulls in a clever way, and the solution could look like this:

{% highlight sql %}
with combine_events as (
  select
    campaign_id,
    event_ts,
    1 as impressions,
    0 as clicks,
    0 as conversions
  from impressions

  union all

  select
    campaign_id,
    event_ts,
    0 as impressions,
    1 as clicks,
    0 as conversions
  from clicks

  union all

  select
    campaign_id,
    event_ts,
    0 as impressions,
    0 as clicks,
    1 as conversions
  from conversions
  where amount > 0
)
select
  user_id,
  event_ts :: date as dt,
  sum(impressions) as impressions,
  sum(clicks) as clicks,
  sum(conversions) as conversions
from combine_events
where dt between '2020-07-01' and '2020-07-19'
group by 1, 2;
{% endhighlight %}

True, it's more verbose, but also much more robust and easier to read. Imagine that you need to extend the code at a later point and add some more events from another source to the query, either more data for an existing event or a new event type. With a `full join` query you'd have to carefully think about how the records behave together. With the `union all` approach you'd simply add a new source block, an extension to the code that could best be described as mechanical.

This post is inspired by the work I did at the Data Systems team at [Essence Digital][essence-digital]. In our specific use case the `union all` solution was not only easier to read and extend, but it also performed better.

[essence-digital]: https://essenceglobal.com/
