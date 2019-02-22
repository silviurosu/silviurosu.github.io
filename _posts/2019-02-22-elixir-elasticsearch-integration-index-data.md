---
layout: post
cover: 'assets/images/elasticsearch-cover.png'
title: Integrate Elixir with Elastisearch - Part 1 - Indexing
description: How to integrate Elixir with Elasticsearch Cluster. How to configure, map the schema and index documents.
date: 2019-02-22 10:00:00
tags: Elixir OTP Elasticsearch
subclass: 'post tag-test tag-content'
categories: Elixir Elasticsearch
navigation: True
comments: true
---

On a recent project I had hard time making Elixir integrate well with Elasticsearch. I decided to write this guide to ease the process for others.

First we need to have a working elasticsearch cluster. I have used [Qbox](https://qbox.io/) for this and I am happy how it works. It's just a matter of clicks to setup a working ES cluster on your chosen server provider.

For the Elixir integration I have used this mix dependency [elasticsearch-elixir](https://github.com/danielberkompas/elasticsearch-elixir). It has no DSL and it's closer to raw communication. There is no more that a few http endpoints to call anyway.

Below is a step by step guide. I will insist more on the parts where I had more difficulty.

Add dependency to your mix file:

```elixir
def deps do
  [
    {:elasticsearch, "~> 0.6.2"}
  ]
end
```

Then we need to create a module for the cluster:

```elixir
defmodule MyApp.ElasticsearchCluster do
  use Elasticsearch.Cluster, otp_app: :my_app
end
```

where **otp_app** is the name of the app where you are integrating this. We need to add Elasticsearch to your supervision tree:

```elixir
children = [
  MyApp.ElasticsearchCluster
]
```

Configuration is done via config.exs file. This is the first part where I had difficulty. Since the staging and prod env are booth loaded from *prod.exs* and many settings are different we need to be able to change the config at runtime. I achieved this by adding config settings for booth staging env and prod in the same *prod.exs* file and I set them on runtime via Application. You can specify env by adding *ELASTICSEARCH_ENV* in your server ENV vars to *staging* or *prod*.

```elixir
config :my_app, ELASTICSEARCH_ENV: "staging"

config :my_app, :elasticsearch_settings_staging,
  username: "username",
  password: "password",
  json_library: Jason,
  url: "https://server_address",
  api: Elasticsearch.API.HTTP,
  indexes: %{
    restaurants_staging: %{
      settings: "/priv/elasticsearch/restaurant.json",
      store: MyApp.ElasticsearchStore,
      sources: [MyApp.RestaurantIntegration],
      bulk_page_size: 10,
      bulk_wait_interval: 5000
    }
  },
  default_options: [
    timeout: 10_000,
    recv_timeout: 5_000,
    hackney: [pool: :elasticsearh_pool]
  ]

config :my_app, :elasticsearch_settings_prod,
  username: "username",
  password: "password",
  json_library: Jason,
  url: "https://server_address",
  api: Elasticsearch.API.HTTP,
  indexes: %{
    restaurants_prod: %{
      settings: "/priv/elasticsearch/restaurant.json",
      store: MyApp.ElasticsearchStore,
      sources: [MyApp.RestaurantIntegration],
      bulk_page_size: 100,
      bulk_wait_interval: 3000
    }
  },
  default_options: [
    timeout: 10_000,
    recv_timeout: 5_000,
    hackney: [pool: :elasticsearh_pool]
  ]

```

Depending of the size of the server you need to play with *bulk_page_size* and *bulk_wait_interval*. Otherwhise it may run out of Java Heap Space very soon and crash. This happened to me and it took me a few days scaling the server and changing this settings to make it work.

Next step, inside the OTP Application *start* callback I added this:

```elixir
defmodule MyApp.Application do
  use Application

  alias MyApp.ElasticsearchCluster

  def start(_type, _args) do
    import Supervisor.Spec, warn: false

    children = [
      ElasticsearchCluster
    ]

    cfg = System.get_env(:ELASTICSEARCH_ENV)

    if cfg do
      Application.put_env(:my_app, :ELASTICSEARCH_ENV, cfg)
    end

    update_elasticsearch_config()

    opts = [strategy: :one_for_one, name: MyApp.Supervisor]
    Supervisor.start_link(children, opts)
  end

  defp update_elasticsearch_config do
    elastic_env = Application.get_env(:my_app, :ELASTICSEARCH_ENV)
    cluster_key = String.to_atom("elasticsearch_settings_#{elastic_env}")
    elastic_settings = Application.get_env(:my_app, cluster_key)
    elastic_settings = update_settings_file_path(elastic_settings, elastic_env)

    Application.put_env(:my_app, MyApp.ElasticsearchCluster, elastic_settings)
  end

  defp update_settings_file_path(conf, elastic_env) do
    path = Path.join(:code.priv_dir(:my_app), "/elasticsearch/restaurant.json")

    put_in(conf, [:indexes, String.to_atom("restaurants_#{elastic_env}"), :settings], path)
  end
end
```

The most difficult part I encountered was finding the path to **settings** file. Since at runtime the path of this file is changed by compilation I had to retrieve it at runtime and set it back in config.


The **restaurants.json** file contains the index mapping. I have added here a snippet from my config as an example:

```json
{
  "mappings": {
    "_doc": {
      "dynamic": false,
      "properties": {
        "permalink": {
          "type": "keyword"
        },
        "name": {
          "type": "text"
        },
        "locale": {
          "type": "keyword"
        },
        "published": {
          "type": "boolean"
        },
        "pause_online_ordering": {
          "type": "boolean"
        },
        "channels": {
          "type": "keyword"
        },
        "created_at": {
          "type": "date"
        },
        "updated_at": {
          "type": "date"
        },
        "configuration": {
          "properties": {
            "accept_pickup": {
              "type": "boolean"
            },
            "accept_delivery": {
              "type": "boolean"
            },
            "accept_curbside": {
              "type": "boolean"
            },
            "accept_dinein": {
              "type": "boolean"
            }
          }
        },
        "cuisines": {
          "type": "keyword"
        },
        "amenities": {
          "type": "keyword"
        },
        "rules": {
          "properties": {
            "zip": {
              "type": "keyword"
            },
            "city": {
              "type": "keyword"
            },
            "shape": {
              "type": "geo_shape",
              "precision": "1m",
              "tree": "quadtree",
              "distance_error_pct": 0.005
            }
          }
        }
      }
    }
  }
}

```

To index the data on staging and production servers I have created a genserver to manage all the indexing part. Since I am running a cluster of nodes I want to have a single running process in the cluster to take care of the index. Otherwhise the Elasticsearch server may not be able to handle the load and it will run out of Java Heap Memory. Below is a snippet from my Genserver:

```elixir
defmodule MyApp.IndexerGenserver do
  use GenServer

  require Logger

  alias Elasticsearch.Cluster.Config
  alias Elasticsearch.Index
  alias Elasticsearch.Index.Bulk
  alias Domain.BO.RestaurantsSearchParams
  alias MyApp.ElasticsearchCluster
  alias MyApp.ElasticsearchStore
  alias MyApp.Exceptions.ElasticsearchError
  alias MyApp.IndexerSupervisor

  alias __MODULE__, as: M

  def child_specs(args) do
    %{
      id: M,
      start: {M, :start_link, args},
      restart: :transient,
      type: :worker
    }
  end

  def start_link(_args) do
    GenServer.start_link(M, [], name: process_name())
  end

  @impl GenServer
  def init(_) do
    {:ok, nil}
  end

  def process_name, do: {:global, M}

  @doc """
  Hot swap index. Zero downtime index rebuilding.
  It created a new index where it sends all the documents then replaces the indexes
  """
  def hot_swap do
    {:ok, pid} = IndexerSupervisor.get_indexer_process()
    GenServer.cast(pid, {:hot_swap})
  end

  @doc """
  Reindex the specified restaurant.
  """
  def index_restaurant(restaurant) do
    {:ok, pid} = IndexerSupervisor.get_indexer_process()
    GenServer.cast(pid, {:index_restaurant, restaurant})
  end

  def handle_cast({:hot_swap}, state) do
    config = Config.get(ElasticsearchCluster)
    alias = String.to_existing_atom(index_name())
    name = Index.build_name(alias)
    %{settings: settings_file} = index_config = config[:indexes][alias]

    with :ok <- Index.create_from_file(config, name, settings_file),
         bulk_upload(config, name, index_config),
         :ok <- Index.alias(config, name, alias),
         :ok <- Index.clean_starting_with(config, alias, 2),
         :ok <- Index.refresh(config, name) do
      :ok
    else
      err ->
        Bugsnag.report(err, severity: "warn")
    end

    {:noreply, state}
  end

  def handle_cast({:index_restaurant, restaurant}, state) do
    %RestaurantsSearchParams{restaurant_id: restaurant}
    |> ElasticsearchStore.stream()
    |> Enum.each(&index_restaurant_data/1)

    {:noreply, state}
  end

  defp bulk_upload(config, name, index_config) do
    case Bulk.upload(config, name, index_config) do
      :ok ->
        :ok

      {:error, errors} = err ->
        Logger.error(fn -> inspect(errors) end)

        Bugsnag.report(
          ElasticsearchError.exception("Errors encountered indexing restaurants"),
          severity: "warn",
          metadata: %{errors: errors}
        )

        err
    end
  end

  defp index_restaurant_data(restaurant) do
    str_id = Integer.to_string(restaurant.id)

    case Elasticsearch.put_document(ElasticsearchCluster, restaurant, index_name()) do
      {:ok, _} ->
        Logger.info(fn -> "Succesfully indexed restaurant #{str_id}." end)

      {:error, %HTTPoison.Error{id: nil, reason: :timeout}} ->
        Logger.error(fn -> "Timeout indexing restaurant #{str_id}" end)

        Bugsnag.report(
          ElasticsearchError.exception("Timeout indexing restaurant #{str_id}"),
          severity: "warn",
          metadata: %{restaurant: restaurant}
        )

      {:error, %Elasticsearch.Exception{message: message, raw: raw_error}} ->
        Logger.error(fn -> message end)
        Logger.error(fn -> inspect(restaurant) end)

        Bugsnag.report(
          ElasticsearchError.exception(message),
          severity: "warn",
          metadata: %{restaurant: restaurant, message: message, elasticsearch_error: raw_error}
        )
    end
  end

  def restaurants_domain do
    Application.get_env(:my_app, :restaurants_domain)
  end

  defp index_name do
    elastic_env = Application.get_env(:my_app, :ELASTICSEARCH_ENV)
    "restaurants_#{elastic_env}"
  end
end

```

To load the data from database I have created a custom stream. Elasticsearch library I have used also requires *Elasticsearch.Cluster* to expose a stream to bulk upload the data. I have lost a bit of time with this so I added my implementation maybe it's useful:

```elixir
defmodule MyApp.ElasticsearchStore do
  @behaviour Elasticsearch.Store

  alias __MODULE__, as: M
  alias Domain.BO.RestaurantsSearchParams

  defstruct [:total, :page, :data]

  @per_page 200

  @params %RestaurantsSearchParams{
    per_page: @per_page
  }

  def base_params, do: @params

  @impl true
  def stream(%RestaurantsSearchParams{} = params) do
    params = %RestaurantsSearchParams{params | per_page: @per_page}
    do_stream(params)
  end

  def stream(_schema) do
    do_stream(@params)
  end

  defp do_stream(params) do
    Stream.resource(
      fn ->
        total = restaurants_domain().count_restaurants_for_index(params)

        params = %RestaurantsSearchParams{params | page: 1}

        data = load_restaurants(params)
        %M{total: total, page: 1, data: data}
      end,
      fn acc ->
        {elem, acc} = extract_next_element(acc, params)

        if is_nil(elem) do
          {:halt, acc}
        else
          {[elem], acc}
        end
      end,
      fn _ -> nil end
    )
  end

  @impl true
  def transaction(fun) do
    fun.()
  end

  defp extract_next_element(%M{data: [elem | tail]} = acc, _params) do
    {elem, %M{acc | data: tail}}
  end

  defp extract_next_element(%M{data: []} = acc, params) do
    %M{total: total, page: page} = acc

    if page * params.per_page >= total do
      {nil, acc}
    else
      next_page = page + 1

      params = %RestaurantsSearchParams{params | page: next_page}

      data = load_restaurants(params)
      [head | tail] = data
      acc = %M{total: total, page: next_page, data: tail}
      {head, acc}
    end
  end

  defp load_restaurants(params) do
    restaurants_domain().restaurants_for_index(params)
  end

  defp restaurants_domain, do: Application.get_env(:my_app, :restaurants_domain)
end
```

Now you should be able to reindex all documents by running
```elixir
MyApp.IndexerGenserver.hot_swap()
```
inside a console to your server.

I'll get back to part two where I'll show how to search documents in this index and how to abstract the Elasticsearch query syntax to one easier to use.



If you need help setting up all this in an Elixir application please [contact me](http://www.ubazu.com/about/)
