---
layout: post
cover: 'assets/images/phoenix.jpg'
title: Strategies to restore GenServer state in Elixir
description: How to restore GenServer state on process restoration.
date: 2018-09-05 10:00:00
tags: Elixir OTP
subclass: 'post tag-test tag-content'
categories: Elixir
navigation: True
comments: true
---

OTP is one of the nicest features of Elixir/ Erlang that convinced me to migrate from Ruby to Elixir for web applications. Finally we have an easy way to make stateless web applications more stateful. Elixir GenServer process can handle requests without the need to reload the whole context from database to know what to do with the request. An Elixir OTP application is more like a an actor base system where the actor is not forgetful, knows the context of discussion, can do work and take decisions outside of the request/response lifecycle. It's more closer to how people work.

With this power comes also a great risk though. What happens when the process dies due to any of this reasons:
 - an error in the code
 - a server restart
 - a redeploy of the application
 - we explicitly terminated the genserver due to inactivity (timeout)

The whole state will be lost. We need a way to resume the process state where it was before.

Some strategies I have in my mind we can use are:

1) **Serialize state for every change**

We can serialize the whole state with a persistence mechanism like a database, an external cache like redis, or even erlang [dets](http://erlang.org/doc/man/dets.html). This is the simplest solution I can see and it may be the best solution if the state is not too big or it does not change too often.

2) **Save incremental changes in database**

We can update database tables on each change in the state. This is the approach that I took on one of my projects. Since the state is distributed in many database tables I can update each small change in there.

An example application can be a shopping cart. When we add an item in the cart we can save in database only the new item added.

3) **Save changes of the state as event sources**

We can use the [event sourcing](https://martinfowler.com/eaaDev/EventSourcing.html) pattern to save state changes as events in database. When we restore cart we can replay those events. We need to be careful though in case the event list becomes too big.


# How to safe call GenServer methods

Since the GenServer it can be live of not we need to check on each call on GenServer that it is still running and restart it otherwise. I used a bit of meta-programming for this in my code. Relevant code is below:

```elixir
  defmacrop safe_process_action(uuid, body) do
    quote do
      if RegistryHelper.process_exist?(@cart_registry, unquote(uuid)) do
        unquote(body)
      else
        case CartDeserializer.load(unquote(uuid)) do
          {:ok, %CartState{} = state} ->
            init_new_cart_process(unquote(uuid), state)
            unquote(body)

          {:error, _} ->
            raise @cart_restore_error <> " " <> unquote(uuid)
        end
      end
    end
  end

  defp init_new_cart_process(id, state) do
    CartSupervisor.new_cart(id, state)
  end

  @impl GenServer
  def init({uuid, %CartState{} = state}) do
    {:ok, state, cart_timeout()}
  end
```

Now I wrap all calls to GenServer in this new method:

```elixir
  def get_cart_content(id) when is_binary(id) do
    safe_process_action(id, GenServer.call(process_id(id), {:get_content}))
  end
```

Since each call to GenServer is wrapped in *safe_process_action* cart restoration is done automatically. If the Registry does not have the reference to GenServer we init a new cart process and then we send the message to it. This guarantees us that in case the cart has timed-out we can restore the cart with state and continue it's work where it left off.

## Further optimizations

Also if we are not sure about the code in GenServer (it can crash often due to business logic of external services) and it's costly to restore the state, we can extract the state in another process that only holds the state but has no business logic and we can request the state from that process when we do our business logic.

Another optimization that we can do is related to Supervisor. We do not want to block the Supervisor until the GenServer was able to restore it's state. If we have thousands of active carts and the application is redeployed, the Supervisor will wait for each GenServer to return from init before starting the other one. We do not want that. We can use the new feature in OTP 21 **handle_continue**. The init method can return an empty state and free up the Supervisor, then *handle_continue* will do the rest of the work to restore the cart. This method guarantees us that any messages to GenServer will be handled after the initialization of the process is finished.
