---
layout: post
cover: 'assets/images/nanobox.jpg'
title: Deploy Elixir Application on Nanobox
description: How to deploy and Elixir based app on nanobox docker infrastructure backed by Amazon Ec2 instances.
date: 2018-03-30 10:00:00
tags: Devops Elixir
subclass: 'post tag-test tag-content'
categories: 'Elixir'
navigation: True
comments: true
---

We recently moved our Elixir based applications to [nanobox](https://nanobox.io/) from heroku. Heroku is built mostly for Ruby on Rails, but for Elixir we found many drawbacks like:

- heroku does daily restarts of the server and we lose all the state
- we do not have any port open on heroku and we can not connect remote observer
- no cluster of nodes can be built on heroku
- multiple cores dyno are very expensive on heroku and we need it for Elixir concurency
- no http2 or websockets support

With nanobox we have:

- whatever server we want from Amanzon with no additional price. Nanobox takes a small fee only for the infrastructure support. We can use compute optomized instances with multiple cores at a fraction of what would cost us on heroku and we can take advantage of Elixir concurency.
- easy deploy process
- we can scale the server any time vertically or horizontally

The only thing that nanobox does not have yet to fully support Elixir magic is hot redeploy. But we do not use that so it's not a problem for us.

Below is a short guide how I deployed my first app on nanobox:

### Create a new nanobox app

  Login in nanobox [dashboard](https://dashboard.nanobox.io/) and create a new app.

### Scale the server to required size

  By default nanobox starts a t2.nano instance that is a bit small for my need. So fell free to scale the instance to a more appropiate size. At this step I usualy have an issue sometimes: nanobox can not connect to newly started EC2 intance and I need to reboot it manually from Amazon dashboard. But this is not a major thing if you know what to do.

### Add boxfile.yml to your app
  This file is required by nanobox to manage the docker containers. My working configuration

```yaml
run.config:
  engine: elixir

  engine.config:
    runtime: elixir-1.6
    erlang_runtime: erlang-20

  cache_dirs:
    - deps

  dev_packages:
    - inotify-tools

  extra_packages:
    - git

  build_triggers:
    - mix.lock

deploy.config:
  after_live:
    web.main:
      - 'curl -X POST --data-urlencode "payload={\"channel\": \"#deploys\", \"username\": \"nanobox\", \"text\": \"$APP_NAME successfully deployed.\", \"icon_emoji\": \":docker:\"}" https://hooks.slack.com/services/xxxxxxxxxxxxx'

web.main:
 start: node-start mix phx.server
 writable_dirs:
    - _build
```

I am using latest Elixir and Erlang version at this time.
Also I have git dependencies so I added git extra package.
I added a hook on after live to notify our slack channel that a new deploy has beed done. Please replace channel url with your own channel.

Application is started using [node-start](https://github.com/nanobox-io/nanobox-engine-elixir/blob/master/files/bin/node-start) script offered by nanobox that adds also the name and the cookie to be able to connect multiple nodes later. This is very usefull for creating a cluster of nodes or connecting Erlang observer.

### Install nanobox on your local machine.
  You can download the package [here](https://dashboard.nanobox.io/download?ci=54798d85-fd9a-4e5c-b3f6-500b4cdc7745)


### Link your application to nanobox server

```shell
nanobox remote add production nanobox-app-name
```

Replace *nanobox-app-name* with the name of your app. I usualy create two servers, one for *staging* and one for *production*.

### Update environment variables

  Go to config and update env variables to your needs.

### Deploy the app

```shell
nanobox deploy production
```

Now you should have a working instance deployed on nanobox. If you encounter issues either ask me either access nanobox slack channel for help.
