---
layout: post
cover: 'assets/images/nanobox.jpg'
title: Connect Erlang Observer to Elixir App deployed on nanobox
description: How to connect Erlang observer to Elixir based app deployed on nanobox.
date: 2018-03-30 10:00:00
tags: Devops Elixir Observer
subclass: 'post tag-test tag-content'
categories: 'Elixir'
navigation: True
comments: true
---

Having an application based on GenServers that maintain state in memory I wanted to be able to connect Erlang Observer to production app to see process tree and debug memory leaks. After a lot of investigation I found a working solution that I will describe below.

First I will start with a fist time config that needs to be done when a new EC2 instance is created, either the app is scalled either a new nanobox app is created.

## First time configuration

1. Get nanobox ssh key using [this guide](https://docs.nanobox.io/live-app-management/remote-access/app-ssh-keys/)

**For each server you will need to get a different key.**

Save this key locally and change file access:

```shell
chmod 600 ~/.ssh/nanobox_key
```

2. Log inside nanobox server with ssh

```shell
ssh ubuntu@server.com -i ~/.ssh/nanobox_key
```

3. Enable GatewayPorts

By default remote port forwarding can only bind to the loopback adapter on the remote host which will not be accessible from a container. To change this we must edit `/etc/ssh/sshd_config` on the server and add this line:

```
GatewayPorts clientspecified
```

Ensure there are no other GatewayPorts lines in the file. Next, reload the configuration by executing `sudo service ssh restart`. (These instructions are for Ubuntu, it might be different on other distributions.)

### Connect local observer

1. Connect inside application docker container

```shell
nanobox console production web.main
```

- *web.main* can be different if the server is scaled horizontally. Please check Nanobox dashboard.
- *production* is the alias I gave in nanobox for the server. Put whatever is you alias in here


2. Run node-attach to connect to running instance

Inside nanobox console run:

```shell
/app $ node-attach
Erlang/OTP 20 [erts-9.2.1] [source] [64-bit] [smp:1:1] [ds:1:1:10] [async-threads:10] [kernel-poll:false]
Interactive Elixir (1.6.3) - press Ctrl+C to exit (type h() ENTER for help)
iex(web-main-6-1@192.168.0.3)1>
```

3. Run a local IEX session

We also start a local IEX session. We need to set the cookie and also the node name. The IP of the name is the Docker bridge interface (docker0) on the server (by default 172.17.0.1) as our remote node will use this to connect.

We also use the inet_dist_listen_min/max parameters to ensure that node listens on port 19000 so we can forward this port.

```shell
my-laptop$ iex --name node@172.17.0.1 --cookie nanobox --erl "-kernel inet_dist_listen_min 19000 inet_dist_listen_max 19000"
iex(node@172.17.0.1)1>
```

4. Forward local ports to the server

Next we create the SSH tunnel and map the local EPMD port and port 19000 onto the Docker bridge IP on the remote server. In another console run:

```shell
ssh ubuntu@server.com -R 172.17.0.1:4369:127.0.0.1:4369 -R 172.17.0.1:19000:127.0.0.1:19000 -N -i ~/.ssh/nanobox_key
```

5. Connect the nodes together

Now we can get the remote node to connect to our local node:

```shell
iex(web-main-6-1@192.168.0.3)2> Node.connect :'node@172.17.0.1'
true
```

The connection is bi-directional so on our local node we can see that our remote node is now connected and we can also start the Observer. Once started, we can select our remote node from the Observer ‘Nodes’ menu.

```shell
iex(node@172.17.0.1)2> Node.list
[:"web-main-6-1@192.168.0.3"]
iex(node@172.17.0.1)3> :observer.start
:ok
```
