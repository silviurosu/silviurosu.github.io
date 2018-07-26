---
layout: post
cover: 'assets/images/nanobox.jpg'
title: Benefits of using structs in Elixir
description: How to use structs to have compile time checking.
date: 2018-07-26 10:00:00
tags: Elixir
subclass: 'post tag-test tag-content'
categories: 'Elixir'
navigation: True
comments: true
---

Elixir structs is one of the things that I tend to use a lot in my code to have a clear set of data to pass around instead of relying on maps and not knowing what I can find inside.

Basically a struct is a special type of map that has a predefined list of keys:

```elixir
iex> defmodule User do
...>   defstruct name: "John", age: 27
...> end
```

Quote from docs:

> Structs provide compile-time guarantees that only the fields (and all of them) defined through defstruct will be allowed to exist in a struct:

```elixir
iex> %User{oops: :field}
** (KeyError) key :oops not found in: %User{age: 27, name: "John"}
```

How I leverage this to have compile time checking in my code:

There are multiple cased where we have a method that receives a map with parameters:

```elixir
def login_user(user) do
   case user_service().login_user(user.email, user.password) do
      :ok -> {:ok, true}
      {:error, _} -> {:error, "Email/password combination is not ok"}
   end
end
```

This method is fine by itself but it fails only at runtime if the *email* or *password* fields are not present in the user parameter. Only with a good suite of tests we can find this bug.

I like very much to have errors during compilation time instead of relying on tests or runtime to find bugs like this. If we pattern match on structs we can spot this errors during compilation phase:

```elixir
def login_user(%User{email: email, password: password}) do
   case user_service().login_user(email, password) do
      :ok -> {:ok, true}
      {:error, _} -> {:error, "Email/password combination is not ok"}
   end
end
```


One other case where we can leverage compile time checking is on updating the structs. I have this pattern often in my code:

```elixir
%{user | name: "Gigel", age: 22}
```

In case the name field is not present inside the struct we'll get an error only at runtime or in a test if we have one. But if we specify the struct we'll get the error directly on compile time:

```elixir
%User{user | name: "Gigel", age: 22}
```

Another example from my production code:

```elixir
  defp convert_item(item) do
    %ShoppingItemBO{
      item_id: item_id,
      name: name,
      menu_name: menu_name,
      menu_id: menu_id,
      quantity: quantity,
      price: price,
      category_name: category_name,
      category_id: category_id
    } = item

    %MenuItemBO{
      category: category_name,
      category_id: category_id,
      id: item_id,
      name: name,
      menu: menu_name,
      menu_id: menu_id,
      quantity: quantity,
      price: Decimal.to_float(price)
    }
  end
```


My suggestions is to use the struct when extracting fields from it and when updating fields. This way we can have an extra set of guarantees that we have not misspelled something.
