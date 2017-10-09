---
layout: post
cover: 'assets/images/cover2.jpg'
title: Why I switched to React from Angular or Knockout
description: Switch from Angular and Knockout to React.js
date: 2015-09-04 10:00:00
tags: React
subclass: 'post tag-test tag-content'
categories: 'React'
navigation: True
comments: true
---

I implemented a lot of web applications using Rails. I find it very easy to work withI like a lot working with client side technologies. Over the year I worked with Jquery, Backbone.js, Knockout, Angular and lately more and more with React. Actually I become a React fan. At the beginning I was sceptical using React mostly because all the view code was mixed with business logic code. It did not seemed elegant at all and I was afraid about separation of concern between developers and designer. With React designed has to enter inside the components code and update JSX code, or the developer has to sync the designed html with his components.

Then because of a colleague of mine that became found of React I started to look more into it and use it inside my applications. Now I like it so much that I do not want to use any other framework for interfaces.

Some of the things that convinced me are:

*it’s very simple to learn

Everything is a component that has 3 base methods: getInitialState, componentDidMount and render. There is almost nothing else to learn. A few things about events and how to compose components and that’s it. You’re ready to go.

*easy to split

Because view is broken in small components it can be easily separated inside the team members that are responsible of their own piece of the interface.

*readability

Html code(jsx) is included inside component code it’s easy to read the code. You have to look at just one source file. This makes the code more maintainable and readable.


* focused

It does take care only of the View part of MVC. It keeps the DOM in sync with the application state. There are no templates and no string concatenations. View code is composed from cals to javascript methods.

* dynamic views
*
View can be very dynamic based on the internal state of the component. The DOM is updated transparent and you do not need to access it directly.

* separation of concerns

Everything is a small component that does just one single thing and no more. It does not know about other parts of the view. And also the business logic code can be extracted in other classes that are not view related.

* automatically view updates

The DOM object is automatically synced with the shadow dom and you do not need to worry about this.

* optimal view updates

The updates on the DOM are done more optimal that with other frameworks. Everything is updated first in shadow dom and then it is synced later with minimal updates possible. This leads to a increase of speed.


* isomorphism

The code can also be rendered on the server side. This is a major help on SEO and mobile performance. Also the client side can take the view from where the server has left and continue from there. The only thing you have to do is to serialise the objects you already have.

I think that React combined with [Baconjs](https://baconjs.github.io/api.html) or any other Functional Reactive Programming library can lead to fast to write and easy to maintain web applications.
