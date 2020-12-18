---
layout: post
cover: 'assets/images/software.jpg'
title: How a good software engineer should approach writing code
description: Steps that a good software engineer should take when writing code.
date: 2020-12-16 10:00:00
tags: Software Engineer
subclass: 'post tag-test tag-content'
categories: 'Software'
navigation: True
comments: true
---

During my career I wrote code in many different ways and learned from mistakes the hard way with issues hitting back when they were at least expected. With time I have developed a personal flow that, in my opinion, it's the right way to approach writing software go get the maximum combination of speed, quality, readability, maintenance, stability and ease to extend that code further.

I have seen many developers with other approaches, taking different shortcuts that on the short term allow a faster release into the wild but from my experience those shortcuts hit back when least expected and for the very long run lead to code that crash often on changes or that can not be extended further and requires a full rewrite.

These are the steps I force myself to take:

1. Ensure the requirements are clear
    Try to understand all the requirements before jumping to write code. Read the specification, if there are none ask questions and make a small list of specifications. Think about if you see something that can not be implemented or that does not have logic and raise questions.

    Skipping this step usually leads to much time lost figuring out the requirements on the fly and changing code again and again when you find new things.

2. Think about the architecture
    Put some serious though into how you want to approach the feature. It there something that you do not know how to do, ask other developers more experienced.

3. How can I test this feature?
    Think about what tests do you need to write to be sure the functionality is working. Try to cover as much edge cases as you can see. Write those tests. You can do it one by one and making each one pass or more tests at once. Try to be as close as possible to TDD practices. Be careful here to test only the public methods, only the entrance to the feature, not expose internals in the test. Also try to test in isolation when you do unit tests. If you call other services already tested you can mock them, if you test business logic you can mock database access, if you have external API calls you can mock those too and so on.

4. Now write the code
    Write the code as clean as you can get from the first shot. Do not worry too much since you can refactor at the end. Use general principles here like single responsibility, pure functions and other practices you have learned.

5. Refactor the code to be more readable
    After writing the code and making the tests pass you can take care of cleaning it up. Read it and see how easy it is to follow the logic, does it requires more documentation, write it. How easy is for other developers to read it, or for me to read it after a few months. Refactor if required to be more clean.

6. Is there a pattern that can be reused in the code
   It is something that can be reused like an utility function, then extract it to an utility module or class.

7. How fast my code runs?
    This is not mandatory but nice to have. See if you can measure the speed of the code. How does it behave on high load. Can I have a benchmark on top

8.  How can I watch the code in production?
    Can I collect some metrics in production to see how it behaves and to be warned when something is not right? Integrate it with your current metrics system (prometheus, grafana, datadog or whatever you have).



    Only now the code can be sent to manual testing team, automate testing of whatever you have in place in the company and then released into the wild.
    Following these steps will help you keep your mental sanity, be proud of your work, sleep well at night and be responsible for your company and the ones who pay you.