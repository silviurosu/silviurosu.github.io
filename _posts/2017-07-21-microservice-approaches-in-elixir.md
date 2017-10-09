---
layout: post
cover: 'assets/images/cover2.jpg'
title: Microservice approaches in Elixir
description: An approach to microservices in Elixir
date: 2017-07-21 10:00:00
tags: Architecture microservices Elixir umbrella
subclass: 'post tag-test tag-content'
categories: 'Elixir'
navigation: True
comments: true
---

Microservices are used to lousily couple services that talk to each other to implement business logic capabilities. Recently this approach is used more and more to separate large, complex applications in smaller ones easier to manage. I will not enumerate all the advantages because there is already a lot of talking about this.

My biggest discontent about them is that usually they communicate to each other through http calls, being deployed on separate servers. When an API needs to call several other API's behind to implement it's business logic it affects very much the total response time. Let's say that medium response time is 300ms, calling 3 microservices adds 1s extra for the initial API call making it very slow.

If we take in consideration that usually the API calls between microservices needs to be authorized, this means one extra call for each in authorization microservice.

And when you use a language like Ruby that is mostly single process you slow down the availability of your service enormous. With 1s+ response time you can not handle too much concurrent traffic even though you scale the number of servers.

Elixir and Phoenix being concurrent and every request being handled in a separate process you can have as many concurrent request as you want. But still the response time is slow due to multiple microservice calls.

I think in Elixir we can think differently and use as tools [umbrella projects](https://elixir-lang.org/getting-started/mix-otp/dependencies-and-umbrella-apps.html#umbrella-projects) and [internal dependencies](https://elixir-lang.org/getting-started/mix-otp/dependencies-and-umbrella-apps.html#internal-dependencies). Umbrella can be used to separate domain models and possibly multiple phoenix apps in the same project. Internal dependencies can be used to extract domain model in a separate repository and reuse it across multiple Elixir applications.

Using this tools we can separate out complex application in microservices with one of this approaches:

## 1. Having a single project separated in multiple smaller apps and domains per umbrella apps

We can have a single big project that is composed from multiple smaller projects separated in [umbrella apps](https://elixir-lang.org/getting-started/mix-otp/dependencies-and-umbrella-apps.html#umbrella-projects). We can also put multiple Phoenix apps inside as umbrella and we can forward the request form the main Phoenix app based on domain where the request comes to if we want also to separate the api in domains.
Each domain model being an umbrella we can call include them in multiple phoenix apps and call them without having to to through Phoenix and http life-cycle.

### Advantages to this approach

- we have all the domain models in the same place and we can use them from any app
- we can deploy all on a single server and vertical scale when we need more concurrency

### Disadvantages

- we are somehow tied to a single technology. If on a later time we want to rewrite one microservice with another technology then we'll need to add http calls between them. So it's not a big drawback
- harder to work in bigger teams, other can mess up with the code from other domains they are not responsible for

## 2. Having multiple projects and reusing domains as internal dependencies

We also can separate the project in multiple separate Elixir/Phoenix applications that are deployed separately. The business logic (domain models) is extracted in mix dependencies as private repos. Like this we can include this domains in each applications and call multiple domains inside a Phoenix app. So we still have for external world separate applications and servers, but between them they communicate via the same core that has access to the same database.

### Advantages

- we can have separate servers and deployments for each app
- we can have a better separation in teams. Others can only read the private repo, not write in it

### Disadvantages

- if we use OTP inside domain models and we keep state other than database then each app will have a different instance of that GenServer

## 3. Having different server nodes that communicate between each other by Erlang inter-node communication

We can have separate application with they internal domain model and still communicate between each other using GenServer API and call genservers from other nodes.

### Advantages

- we can have totally independent servers and applications that still communicate between each other through Erlang rpc instead of http calls

### Disadvantages

- we are tied up with Erlang and Elixir, we can not change later one microservice to another language


So in conclusion I would use one of this approaches when trying to separate a complex app in smaller units in Elixir instead or using traditional microservice architecture with http communication. If you have other ideeas please let me know.

