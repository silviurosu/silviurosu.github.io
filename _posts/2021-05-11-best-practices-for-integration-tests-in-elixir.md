---
layout: post
cover: 'assets/images/integration-testing.png'
title: Best practices for integration tests in Elixir
description: Some of the practices I use to write integration tests in Elixir
date: 2021-05-11 10:00:00
tags: Elixir OTP ExUnit Testing
subclass: 'post tag-test tag-content'
categories: Elixir Testing
navigation: true
comments: true
---

Good software requires proper testing. Writing code blind without proper unit test and integration test are never a shortcut, on the long term it leads to slower and slower progress until the project becomes a disaster. I learned this lesson hard way many times. Testing should be on the same level of importance as the code itself.

Also no matter as many unit test you write, you also need to see that all the pieces are working properly when put together. Many times testing everything together it's so hard to do or becomes a challenge leading many developers not to do it on manually testing everything on staging. I also had some challenges to overcome when I tested my code. These are some suggestions taken from what I use to write my integration tests. Feel free to use what you need.

## Switch between mocking external calls and real calls

Test should be fast, very fast, otherwise the whole development process is slowed down. When writing integration test you have to call many external API's that usually are slow making the whole tests scenarios taking tens of minutes. This lead to very slow feedback from continuous integration and very slow deployment. Developers will never run tests locally because they are taking so long.
To overcome this I usually use an environment variable, let's say "MOCK_EXTERNAL_CALLS". Each test should take this in consideration and based on it it should do real calls or not. Once a month I run all the test with real external calls to see that nothing has been changed in external API's but usually the tests are run with a record of the real call. This trick leads to very fast tests.

```elixir
 setup do
    mock_external_calls = System.get_env("MOCK_EXTERNAL_CALLS", "false") == "true"
    {:ok, mock_external_calls: mock_external_calls}
  end
  .....
  test "test 1", %{mock_external_calls: mock_external_calls} do
    if mock_external_calls do
        # add bypass assertions
        .....
    end
    .....
  end
```

## Use bypass to mock external requests
External request are mocked using Bypass or any other library. The response can be captured with a real call when writing the test and them mocked.


```elixir
 setup do
    bypass = Bypass.open()
    {:ok, bypass: bypass}
  end
  .....
  test "test 1", %{bypass: bypass} do
    Bypass.expect_once(bypass, "POST", "path", fn conn ->
        Conn.resp(conn, 200, @payment_methods_response)
    end)
    .....
  end
```

## Switch between real external urls and bypass url
In each test we can switch between real url and bypass one using dependency injection via config values. Also the changes need to be reverted after the test.

```elixir
 setup do
    adyen_checkout_url = Application.get_env(:adyen, :ADYEN_CHECKOUT_URL)
    on_exit(fn ->
      Application.put_env(:adyen, :ADYEN_CHECKOUT_URL, adyen_checkout_url)
    end)
    {:ok, conn: put_req_header(build_conn(), "accept", "application/json")}
  end
  .....
  test "test 1" do
    Application.put_env(:adyen, :ADYEN_CHECKOUT_URL, "http://localhost:#{bypass.port}")
    # write the test
    .....
  end
```

## Manually call webhooks if those are required

If business logic required external webhooks to be called you can take the real request and use it to manually call the webhook.

```elixir
  .....
  test "test 1", %{bypass: bypass} do
    build_conn()
      |> put_req_header("accept", "application/json")
      |> put_req_header("authorization", "Basic " <> Base.encode64("adyen:xxxxxxxxxx"))
      |> dispatch(
        Endpoint,
        :post,
        PaymentsRoutes.v5_adyen_callback_path(conn, :execute),
        @capture_callback_params
      )
    .....
  end
```

## Drain background workers if those are involved in the test

If business logic requires some background workers to be completed to assert the result you can drain the background tasks like this before doing the assert:

```elixir
    Oban.drain_queue(ObanPaymentsService, queue: :adyen_callback)
```
