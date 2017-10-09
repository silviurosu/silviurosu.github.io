---
layout: post
cover: 'assets/images/cover7.jpg'
title: Decouple your Elixir application from Phoenix and Ecto
description: Separate your app from Phoenix and Ecto. Your application logic should not depend on any framework. Business logic should be easy to extract and reuse in any other application.
date: 2017-08-06 10:00:00
tags: Architecture Elixir umbrella
subclass: 'post tag-test tag-content'
categories: 'Elixir'
navigation: True
comments: true
---

I came to Elixir and Phoenix from a Ruby and Rails background. I think Rails applications (most of them that follow standard way) have a lot of messy code. Rails has a good architecture for a web framework but does not promote good practices for writing a web application. Everything is tied up with Rails, a lot of business logic and ActiveResource related code goes into models (fat models approach), controllers and into views so everything becomes tied up with the framework, becomes highly coupled. There is no clear separation between Rails and business logic code.

By default there is no service layer that extracts business logic from models and controllers. Because of this it's hard to write tests. You need to test the whole stack, involving http requests and database access. As application grows tests tend to become very time consuming to run and you may give up on them because they are hard to do.

Coming to Elixir and Phonenix I encountered the same problem at the beginning. There are no standardized practices suggesting separation between Phoenix and business logic. With the coming of Phoenix 1.3 things changed. **_Web_** folder has been moved to **_lib_**, repo has been extracted from **_web_** and **_models_** folder is gone. There are a lot of changes in the right direction. Ecto and business logic is extracted from web and it is suggested that your application should be separate from the Phoenix framework. But still I think it's not enough, Ecto is still coupled with your business logic.

There are a few rules that I keep in mind when writing web applications:
- Phoenix is not your application (it's just a delivery mechanism)
- Ecto is not your application (it's just a data provider and a storage mechanism)
- An application is a set of business logic and rules that can be extracted and reused in any other app
- Application intent should be visible on the first look at the code

For example taking a look at a house plan you know exactly what the plan is for and what is composed of (bedroom, bathroom, kitchen, living-room). But looking at a standard Phoenix app the intent is not so clear any more. Always you will see the standard structure, controller, model, views, router. The only thing that it says is that you have a web application but not what it does.

    ├── application_name
    │   ├── config
    │   ├── controllers
    │   │   └── pages.ex
    │   ├── models
    │   ├── router.ex
    │   ├── supervisor.ex
    │   ├── templates
    │   ├── views
    │   └── views.ex
    └── application_name.ex

Following this principles I suggest using this approach:

    └── application_name
        ├── config
        └── apps
            ├── api
            ├── service
            └── db

Each layer of the application can be separated in a separate umbrella application for clear differentiation. Then in each app I suggest to make a folder structure based on the business logic not on the class types.

![Web Application Design UML](/assets/images/decouple-your-elixir-application-from-phoenix-and-ecto/application-design.jpg)

Where:
 - **_Entity_** represents a plain object, a struct, a data holder
 - **_Service_** encapsulates the business logic. This is the core of the application
 - **_Entity Gateway_** is the interface specification for loading data from DB or any other storage. It does not return database schema, only _Entity_ structs
 - Delivery mechanism (Phoenix) knows only the interface specifications for the services
 - Communication between Phoenix and application is done via plain objects (**_Request Model_** and **_Response Model_**)

## Advantages adopting an architecture like this:
1. clear separation from Phoenix and Ecto

    Application is not coupled with Phoenix and Ecto. It's framework agnostic. It can be extracted to a hex package or private repo and used with any data provider or communication channel.

2. faster tests

    Since business logic does not depend on any framework, tests do not have to go through http life-cycle or database.

3. Modules can be developed separately

    Since the contract is well defined every service can be implemented separately. Also the web communication part and data provider from DB can be developed separately. This makes team work easier.

4. Modules can be reused

    The whole business logic, or separate services can be easily reused in other applications

5. Phoenix and Ecto can be easily be replaced

    Web communication and data provider can be replaced with something else later without changes in the application

6. easier to maintain for long term

    During an application life-cycle almost 80% is maintenance. Services are small and low coupled, easier to test and maintain for long term

7. care for the developers that will work on the project

    Most developers are scared or decline to work on old projects mostly due to the bad architecture and bad code. A good architecture increases the chance that you will find developers to maintain an application and add new features.


## Some disadvantages that some may consider

1. more time to write code

    Good architecture and good practices means that you have to think more and be more careful when writing code. This can increase the time until you release the first version. But for longer term this time will make future code easier to write and maintain, so the total time for the project will be less that code without attention to architecture.

2. more code to write

    Since modules are smaller, low coupled with behaviors, structs, documentation it will make the code base larger. But much more important is readability, maintainability, stability, re-usability.
