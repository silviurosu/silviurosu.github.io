---
layout: post
cover: 'assets/images/cover7.jpg'
title: Using PostgreSQL Jsonb columns in Ecto
description: Using postgresql jsonb columns types in Ecto. A better alternative to mongodb.
date: 2017-11-01 10:00:00
tags: Ecto PostgreSQL Elixir JSONB
subclass: 'post tag-test tag-content'
categories: 'Elixir'
navigation: True
comments: true
---

Postgres introduced recently, in 9.4, _jsonb_ column type, additional to existing json column that already existed before. **JSON** colums is no more than a blob field where the json is dumped as string. **JSONB** on the other fact is stored in a custom format optimized for json operations alowing for more operators. See [stackoverflow thread](https://stackoverflow.com/questions/39637370/difference-between-json-and-jsonb-in-postgres) for more information.

I created a [github repo](https://github.com/silviurosu/ecto-jsonb-example) to showcase how to use it in Ecto. To see it in action clone the repo then create and populate database with 1M records:

{% highlight bash %}
mix ecto.create
mix ecto.migrate
mix run priv/repo/seeds.exs
{% endhighlight %}

I used jsonb to serialize a map in _settings_ field and an array in _acls_ field. Also to showcase that we can create also indexes on this fields I added _settings_index_ and _acls_index_ fields.

In _settings_ field I stored objects that look like this:

{% highlight json %}
{
  "roles": ["admin", "guest", "config"],
  "loyalties": 200,
  "providers": ["google", "facebook", "twitter"]
}
{% endhighlight %}

Now that we have the database ready we can run queries on it inside the jsonb field:

#### String exist as array element:

{% highlight elixir %}
query = from u in User,
        where: fragment("(settings_index->'roles')::jsonb \\? ?", "admin")

result = Repo.all(query)
{% endhighlight %}

#### Array contains value:

We'll use array intersection here.

{% highlight elixir %}
query = from u in User,
        where: fragment("acls_index @> ?", ^[101])

roles_result = Repo.all(query)
{% endhighlight %}

#### Object contains another object:

{% highlight elixir %}
query = from u in User,
        where: fragment("(settings_index)::jsonb @> ?::jsonb", %{loyalties: 100})

result = Repo.all(query)
{% endhighlight %}


#### Object key exists:

{% highlight elixir %}
query = from u in User,
        where: fragment("(settings_index)::jsonb \\? ?", "personal_info")

result = Repo.all(query)
{% endhighlight %}

### Index performance

Also I tested how fast the queries work with and without index.

{% highlight elixir %}
IO.puts "Roles search benchmark with index vs without index"
index_role_query = fn ->
  query = from u in User,
          where: fragment("(settings_index->'roles')::jsonb \\? ?", "admin")
  Repo.all(query)
end

no_index_role_query = fn ->
  query = from u in User,
          where: fragment("(settings->'roles')::jsonb \\? ?", "admin")
  Repo.all(query)
end

Benchee.run(%{
  "with index"    => index_role_query,
  "without index" => no_index_role_query
}, time: 10)
{% endhighlight %}

This are my results:

{% highlight bash %}
Benchmarking with index...
Benchmarking without index...

Name                    ips        average  deviation         median         99th %
with index           2.94 K        0.34 ms    ±29.07%        0.33 ms        0.51 ms
without index     0.00734 K      136.22 ms     ±2.00%      135.63 ms      147.23 ms

Comparison:
with index           2.94 K
without index     0.00734 K - 401.06x slower
{% endhighlight %}

As you can see there is significant performance increase if we add proper indexes.
