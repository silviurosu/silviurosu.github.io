---
layout: post
cover: 'assets/images/rabbitmq.png'
title: My way to safely consume RabbitMQ messages from Elixir
description: How to consume RabbitMQ messages without losing any message and not flooding issue tracker in case there is a crash in the code.
date: 2020-09-11 10:00:00
tags: Elixir OTP RabbitMQ
subclass: 'post tag-test tag-content'
categories: Elixir RabbitMQ
navigation: true
comments: true
---

Communication via async messages is the best way to communicate between micro-services. Async communication ensures that you can recover easily from a small downtime or a redeploy without loosing messages.

In our company we use RabbitMQ for message passing between our micro-services written mostly in Elixir. For Elixir there is a very nice and useful abstraction for consuming and acknowledging messages from RabbitMQ called [Broadway](https://hexdocs.pm/broadway/Broadway.html)

Broadway ensures you that the messages are safely handled and acknowledged or marked as failed in RabbitMQ if there is a crash by wrapping your *handle_message* code with rescue.
In my experience I faced a major issue though: In case there is a crash in the business logic code that handles the message it will consume the same message forever and, if you have a bug tracker, will flood that with messages reaching the paid plan limit in an instant. Also in case the problem may be with the current message the next messages from the queue will not be processed any more. I was lucky do discover this in the staging environment and not reaching production.

After a bit of brainstorming my solution to this challenge was to create another queue for failed messages. Let's call initial queue **messages** and the failed queue **failed_messages**. I have setup two Broadway consumers that each consumes from the corresponding queue. The __handle_message__ function for **messages** queue rescues any crash and moves the message into **failed_messages** queue, then acknowledges the message as success. This way the messages from **messages** queue will be further processed.

Now inside **failed_messages** consumer I resque again the crashes, I send it to Bugsang and then I pause for a time that is increased for each crash to not flood the Bugsnag. This way the messages will reach Bugsang only once each 5 minutes for example. Later when you deploy the code fix they will be consumed from this **failed_messages** queue and you will not loose messages.

I have created a macro that I use in all my application consumers. I use it like this:


```elixir
defmodule OutcomesConsumer do
  use Utils.BroadwayConsumer,
    exchange: &Utils.RabbitMQUtils.exchange/0,
    queue: &Utils.RabbitMQUtils.outcomes_queue/0,
    failed_queue: &Utils.RabbitMQUtils.outcomes_failed_messages_queue/0,
    failed_queue_routing_key: &Utils.RabbitMQUtils.outcomes_failed_messages_routing_key/0,
    prefetch_count: 6,
    processor_max_demand: 6,
    processor_concurrency: 3

  @impl true
  def handle_message(string_payload) do
    data = Jason.decode!(string_payload)

    # handle message business logic
  end
end
```

And my version of the macro is below. I striped docs and what was not relevant:

```elixir
defmodule Utils.BroadwayConsumer do
  defmacro __using__(options) do
    quote location: :keep, bind_quoted: [options: options, module: __CALLER__.module] do
      @behaviour Utils.RabbitMqBehaviour

      Kernel.defmodule ConsumerError do
        defexception message: "Rabbit Comsumer error"
      end

      Kernel.defmodule MessageConsumer do
        @moduledoc false
        use Broadway

        alias Broadway.Message
        alias Broadway.Options
        alias Utils.RabbitMQHelper
        alias Utils.RabbitMQUtils

        @validation_rules [
          exchange: [required: true, type: {:fun, 0}],
          queue: [required: true, type: {:fun, 0}],
          failed_queue: [required: true, type: {:fun, 0}],
          failed_queue_routing_key: [required: true, type: {:fun, 0}],
          prefetch_count: [required: true, type: :pos_integer],
          producer_concurrency: [required: true, type: :pos_integer],
          processor_min_demand: [required: true, type: :pos_integer],
          processor_max_demand: [required: true, type: :pos_integer],
          processor_concurrency: [required: true, type: :pos_integer]
        ]

        @defaults [
          prefetch_count: 1,
          producer_concurrency: 1,
          processor_min_demand: 1,
          processor_max_demand: 2,
          processor_concurrency: 2
        ]

        def start_link(_) do
          # I validate the parameters received to be sure everything is available
          {:ok, opts} = @defaults |> Keyword.merge(unquote(options)) |> Options.validate(@validation_rules)

          queue_fct = Keyword.get(opts, :queue)
          queue = queue_fct.()
          prefetch_count = Keyword.get(opts, :prefetch_count)
          producer_concurrency = Keyword.get(opts, :producer_concurrency)
          processor_min_demand = Keyword.get(opts, :processor_min_demand)
          processor_max_demand = Keyword.get(opts, :processor_max_demand)
          processor_concurrency = Keyword.get(opts, :processor_concurrency)

         # I start the consumer for messages in the initial queue
          Broadway.start_link(__MODULE__,
            name: __MODULE__,
            resubscribe_interval: 4_000,
            producer: [
              module:
                {BroadwayRabbitMQ.Producer,
                 queue: queue,
                 qos: [
                   prefetch_count: prefetch_count
                 ],
                 connection: RabbitMQUtils.rabbitmq_url()},
              concurrency: producer_concurrency
            ],
            processors: [
              default: [
                min_demand: processor_min_demand,
                max_demand: processor_max_demand,
                concurrency: processor_concurrency
              ]
            ]
          )
        end

        @impl true
        def handle_message(_, %Message{data: string_payload} = message, _) do
          apply(unquote(module), :handle_message, [string_payload])
          message
        # I rescue any crash and move the message to the other queue
        rescue
          e ->
            opts = unquote(options)

            exchange_fct = Keyword.get(opts, :exchange)
            exchange = exchange_fct.()
            failed_queue_fct = Keyword.get(opts, :failed_queue)
            failed_queue = failed_queue_fct.()
            failed_queue_routing_key_fct = Keyword.get(opts, :failed_queue_routing_key)
            failed_queue_routing_key = failed_queue_routing_key_fct.()

            RabbitMQHelper.publish_on_new_channel(
              exchange,
              failed_queue_routing_key,
              string_payload
            )

            message
        end
      end

      Kernel.defmodule FailedMessageConsumer do
        use Broadway

        alias Broadway.Message
        alias Broadway.Options
        alias Utils.RabbitMQUtils

        def start_link(_) do
          opts = unquote(options)

          failed_queue_fct = Keyword.get(opts, :failed_queue)
          failed_queue = failed_queue_fct.()

          # start Broadway to consume messaged from failed consumer
          Broadway.start_link(__MODULE__,
            name: __MODULE__,
            resubscribe_interval: 8_000,
            producer: [
              module:
                {BroadwayRabbitMQ.Producer,
                 queue: failed_queue,
                 qos: [
                   prefetch_count: 1
                 ],
                 connection: RabbitMQUtils.rabbitmq_url()},
              concurrency: 1
            ],
            processors: [
              default: [
                min_demand: 1,
                max_demand: 2,
                concurrency: 1
              ]
            ]
          )
        end

        @impl true
        def handle_message(_, %Message{data: string_payload} = message, _) do
          opts = unquote(options)

          queue_fct = Keyword.get(opts, :queue)
          queue = queue_fct.()
          failed_queue_fct = Keyword.get(opts, :failed_queue)
          failed_queue = failed_queue_fct.()
          failed_queue_routing_key_fct = Keyword.get(opts, :failed_queue_routing_key)
          failed_queue_routing_key = failed_queue_routing_key_fct.()

          apply(unquote(module), :handle_message, [string_payload])
          # reset rate limiting to default value if message successfully processed
          Cachex.put(:failed_messages_rate_limit, failed_queue_routing_key, 1)

          message
        rescue
          e ->
            Bugsnag.report(%ConsumerError{},
              severity: "error",
              stacktrace: __STACKTRACE__,
              metadata: %{message: inspect(message), error: inspect(e)}
            )

            opts = unquote(options)
            failed_queue_routing_key_fct = Keyword.get(opts, :failed_queue_routing_key)
            failed_queue_routing_key = failed_queue_routing_key_fct.()

            {:ok, current_timeout} = Cachex.get(:zuppler_utils, failed_queue_routing_key)
            current_timeout = current_timeout || 1
            Cachex.put(:zuppler_utils, failed_queue_routing_key, min(current_timeout * 2, 5 * 60 * 1_000))

            # sleep current process with timer to not flood the CPU and bug tracker
            :timer.sleep(current_timeout)
            reraise(e, __STACKTRACE__)
        end
      end

      def start_link do
        children = [
          unquote(module).MessageConsumer,
          unquote(module).FailedMessageConsumer
        ]

        Supervisor.start_link(children, strategy: :one_for_one)
      end
    end
  end
end
```

This mechanism ensures me that no message is lost and in case of crashes, after deploying the fix all the pending messages are consumed.
I would love if something like this can be embedded inside Broadway itself to not require additional code to be written by developers.
