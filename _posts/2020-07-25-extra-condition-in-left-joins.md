---
layout: post
title:  "Extra condition in left joins"
date:   2020-07-25
categories: sql
---
Doing a standard left join is straight-forward. But sometimes you need an extra filter on the outer / right-hand table, and there's a small catch that I'm going to present in this entry.

## Standard left outer join

In this example we want to fetch all users and their total invoice amount. If the user has no invoices we return zero for her. Simple enough, just do `users left join invoices`:

{% highlight sql %}
with users as (
  select 1 as user_id
  union all
  select 2 as user_id
), invoices as (
  select 1 as user_id, 100 as amount
  union all
  select 1 as user_id, 300 as amount
)
select
  user_id,
  coalesce(sum(amount), 0) as total_amount
from users
  left join invoices using (user_id)
group by 1;
{% endhighlight %}

The query would return one record for both users: the `total_amount` of 400 for the user 1, zero for the user 2.

## Extra filter on invoices - correct approach

What if the invoices can be refunded and we want to exclude those? Say there's a boolean column `is_refunded` and we want to filter out invoices with `true` in that column. I'm showing the correct approach right away because that's what I'm after in this entry. Later I'll show two typical mistakes for this type of query.

The correct query:

{% highlight sql %}
with users as (
  select 1 as user_id
  union all
  select 2 as user_id
), invoices as (
  select 1 as user_id, 100 as amount, false as is_refunded
  union all
  select 1 as user_id, 300 as amount, true as is_refunded
)
select
  users.user_id,
  coalesce(sum(amount), 0) as total_amount
from users
  left join invoices on users.user_id = invoices.user_id
    and not is_refunded
group by 1;
{% endhighlight %}

This returns the expected results: 100 for the user 1 and zero for the user 2.
The following changes were done:

1. Add the condition `not is_refunded` in the `left join`. This is the key, treat it as another join condition!
2. Because of the second condition, change `using (user_id)` to `on users.user_id = invoices.user_id` 🤷‍♂️
3. And naturally, you now have to specify `users.user_id` in the select

## Typical errors

Two errors are frequent when there's an extra condition like the one we just added to the `invoices` table.

First error is to add the extra condition to a `where` clause, such as `where not is_refunded`. Unfortunately, this messes up our results completely! Remember it's a left join, and we're introducing a condition in `where`, which will be applied to the results **after** executing the join. In effect, our join becomes an inner join! We would be given results only for the user 1, and no record whatsoever for the user 2. This is definitely not what we wanted!

Another common error gives the right results but is hacky. This consist of adding the extra condition in `where` but allowing it to be null, for example `where not is_refunded or is_refunded is null` or `where not coalesce(is_refunded, false)`. Whoever reads the code **might** understand what it does, but it's not the right way to do it nor pretty, so please avoid this approach.

Every day is a school day, and I first learned about this subtle issue with left joins while working at [ClearPeaks][clearpeaks]. After having implemented it several times the hacky way, I finally learned to write my left joins properly.

[clearpeaks]: https://www.clearpeaks.com/
