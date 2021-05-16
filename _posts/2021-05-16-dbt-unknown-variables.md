---
layout: post
title:  "Use dbt variables for Unknown values"
date:   2021-05-16
categories: dbt
---
Default values for "Unknown" and similar might get unwieldy and inconsistent in your data warehouse. We just decided to start using dbt variables instead of hard-coded values.

We first defined a [vars][vars] block with some default values in our `dbt_project.yml`, for example:

{% highlight yaml %}
vars:
  unknown_initcap: "Unknown"
  unknown_lower: "unknown"
  not_applicable: "N/A"
{% endhighlight %}

We'd then switch to using a variable instead of a hard-coded value everywhere in our project.

In models:


{% highlight sql %}
{% raw %}
select
  coalesce(plan, '{{ var("unknown_initcap") }}') as plan,
  ...
{% endraw %}
{% endhighlight %}

In schema tests:

{% highlight yaml %}
{% raw %}
models:
  - name: orders
    columns:
      - name: plan
        tests:
          - not_null
          - accepted_values:
              values: [
                'Professional',
                'Community',
                '{{ var("unknown_initcap") }}'
              ]
{% endraw %}
{% endhighlight %}

Let's see how it goes and whether it was a good idea!

[vars]: https://docs.getdbt.com/reference/dbt-jinja-functions/var
