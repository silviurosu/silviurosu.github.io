---
layout: post
cover: 'assets/images/cover7.jpg'
title: Domain Driven Design with Elixir
description: Separate Elixir application in different domains and extract an interface to be used for each implementation of the domain. Define a common module for accessing this domains
date: 2017-08-09 10:00:00
tags: Architecture DDD Elixir umbrella
subclass: 'post tag-test tag-content'
categories: 'Elixir'
navigation: True
comments: true
---

Im my [previous article](http://www.ubazu.com/2017/08/06/decouple-your-elixir-application-from-phoenix-and-ecto/) I wrote how to decouple your application from Phoenix and Elixir.

In this article I will talk more about separating application in different domains based on the relation between models. So we can further separate db umbrella app from previous post further in more domains.

## Domain driven design

I suggest using DDD to separate domain models in smaller umbrella applications. Usually in an application you can separate some island of entities that are tied to each other and have meaning together. This can become you domain.

As an example let's presume that we have an online ordering system for restaurants. We can separate the database entities in 3 different domain models like this:

![DB separation in domains](/assets/images/domain_driven_design_with_elixir/ddd-models-separation.jpg)

In our application restaurant, user and shopping can become 3 different applications extracted as umbrella apps for example. I created a [small github project](https://github.com/silviurosu/elixir-umbrella-ddd.git) to demonstrate my approach. I separated the app in more umbrella projects like this:

    ├── apps
    │   ├── api
    │   ├── restaurants_db
    │   ├── restaurants_db_behaviour
    │   ├── service
    │   ├── shopping_db
    │   ├── shopping_db_behaviour
    │   ├── users_db
    │   └── users_db_behaviour

 - **api** is the web interface (Phoenix) that talks with the service
 - **service** is the business logic for the application. It contains mostly all the logic. It uses domain models (restaurant, shopping, user)

 - **restaurants_db_behaviour** is the restaurants database behavior specifications that need to be implemented. It also contains the structs that will be used to pass and return the data.
 - **restaurants_db** is the database gateway implementation for restaurant domain model
 - **shopping_db_behaviour** is the restaurants database behavior specifications that need to be implemented. It also contains the structs that will be used to pass and return the data
 - **shopping_db** is the database gateway implementation for shopping domain model


I have 2 applications for each domain. One is the behaviors specifications and the struct (data) that will be pass during communication. I extracted them in a different app because each implementation needs to include this package and implement it. So all the implementation will be plugins that implement this app.

Also I suggest using a single interface file to be used. Do not let users access any model path from your app. For example in _restaurants_db_ we will have:

    ├── restaurants_db
    │   ├── lib
    │   │    ├── restaurants_db.ex
    │   │    ├── load_restaurant
    │   │    │          ├── db_gateway.ex
    │   │    ├── search_restaurant
    │   │    │          ├── db_gateway.ex
    │   │    ├── load_restaurant_menu
    │   │    │          ├── db_gateway.ex
    │

The only model used from applications that talk to restaurants_db application will be **_RestaurantsDB_**.
Inside this file we can delegate the calls to the corresponding models like this:

{% highlight elixir %}
defmodule RestaurantsDB do
  @moduledoc """
   External interface methods for restaurants domain model
  """
  @behaviour RestaurantsDBBehaviour

  alias RestaurantsDB.LoadRestaurant
  alias RestaurantsDB.SearchRestaurant
  alias RestaurantsDB.LoadRestaurantMenu

  @impl true
  defdelegate load_restaurant(restaurant_id), to: LoadRestaurant.DbGateway, as: :load

  @impl true
  defdelegate search_restaurant(name), to: SearchRestaurant.DbGateway, as: :search

  @impl true
  defdelegate search_menu(restaurant_id), to: LoadRestaurantMenu.DbGateway, as: :load
end

{% endhighlight %}

This way we can keep flexible the internal structure on the application without affecting consuming apps.


Having this separation into domains bring a lot of flexibility and advantages like:

- _separating database in multiple smaller ones_. Each domain can have it's own separate db
- _splitting responsibility between team or team members_. Each team can be responsible of a separate domain
- _domains can be easily be reused in other applications_. Either by git submodules, either by extracting them to separate hex packages. I tend to prefer the git submodule approach.
