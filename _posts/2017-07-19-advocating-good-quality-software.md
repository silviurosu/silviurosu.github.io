---
layout: post
cover: 'assets/images/cover2.jpg'
title: Advocating good quality software
description: Advocating the critical importance of Good quality software
date: 2017-07-19 10:00:00
tags: Architecture Elixir TDD quality
subclass: 'post tag-test tag-content'
categories: 'Elixir'
navigation: True
comments: true
---

  It is said that as long as you live you learn and become wiser. The same thing applies to software development. Each of us had this revelation in our experience: we look into the code we wrote 1 year ago and we realize how bad developers we were. It happend to me many times and I think to you also.

  We often tend to write software fast to finish the job and we are eager to go to the next application. We think that writing code is one time thing that we throw into the wild and we forget about it. But it is never like that. During the life of a software application almost 80% is maintainance. Then why we focus so much on the first 20% and we do not care about the rest?

  There can be many answers like:
  - managers are pushing hard
  - deadlines are knocking on the door
  - cost needs to be minimized
  - writing test is boring
  - we want to try the new framework or the new language we read about.

  We think that trying many languages and frameworks makes us better developers, the we improve our CV and according we have a better carreer. But it's not like that. Being a good programmer does not mean to learn many new frameworks. It means to desing a good architecture, to write tests, to write readable and easy to maintain code. The language or the framework we work with is just a detail. Building blocks are the same.

  In this process we forget to be responsible, to be profesionals, to write the code we want to read on others and to respect the other programmers that will read our code later. We do not realize that badly written code will make furter development harder and harder. The overal time to finish the project will be longer and we may regret that we were in a hurry at the begining.

  Why nobody wants to work on legacy code or an old project? Because most of the time is crap. It's almost impossible to read or to maintain it. Often projects can get to the point where the entire application needs to be rewritten from scratch. It happened a few times in my career.

  Another mistake I done in my past carreer and I always saw it happen is lack of tests or tests written for code coverage instead of testing business rules. I adopted TDD and I think is the only way to keep the quality and the stability of the project in hand. When you use TDD you have a lot of advantages like:
   - fun to write tests, they are part of the software not a burder after the code is done just for code coverage or because somebody forced us to have some tests.
   - we are sure that the code compiled and worked a few minutes ago. If we do something wrong we can always reset the changes.
   - we are forced to decouple the code in smaller units to be able to test. Highly coupled code is very hard to test.
   - we start using more generics like intefaces to link the pieces together to be able to inject a test version of the dependency. We do not need to use mock libraries any more.
   - we think for about the business rules of the method we write and we try to cover all the posibilities.

   So in this process we tend to have a better architecture, less coupled code and smaller methods.

  Software should be written with the people in mind, other developers, not machines. Machines can digest almost anything that compiles, people do not. Let's write code that we are proud of and that others may enjoy reading.




