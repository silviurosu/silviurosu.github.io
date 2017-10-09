---
layout: post
cover: 'assets/images/cover2.jpg'
title: How to isolate client side React Single Page Applications from Rails
description: Isolate client side react SPA from Rails
date: 2016-09-19 10:00:00
tags: Architecture Rails SPA React
subclass: 'post tag-test tag-content'
categories: 'Rails'
navigation: True
comments: true
---

I implemented a lot of web applications using Rails. I find it very easy to work with Ruby and Rails to create API and write server business logic. For the client side though I do not like to depend so much on Rails and I prefer to build interactive Single Page Application where the communication with server side is done via API. For this use case I find Rails sprockets not appropriate and not easy to use. It’s not ready for ES6/ES7 also. Although there are gems that can convert from ES6 to ES5 like this one sprockets-es6 it does not support node module system and packaging like webpack does.

In Rails you need to expose in global scope the react components and to make sure they are loaded in the order you access them.
So I prefer to do not use Rails for javascript and bundle it with webpack. This way the client side can be extracted any time separately and server build with any other technology of your choice.

The reason I keep JS application inside Rails and do not deploy it separately as a standalone application only because of the authentication. I do not see any good solution to protect a SPA application besides using oauth2 via hello.js with client side authentication. So I usually keep client application inside Rails and protected them with Devise and oauth2. Authentication is done by rails, if the user is not authenticated the client side application is not loaded at all.

In this post I explain how to configure a new rails application to work as above.

First we need to add webpack-assets gem in Gemfile.

Then we need to create a webpack.config.js file inside Rails root or generate it with this command:

{% highlight shell %}
  bin/rails g webpack_assets:init
{% endhighlight %}

This is my configuration file I use in current project:

{% highlight javascript %}
  var path = require('path');
  var webpack = require('webpack');
  var isProduction = 'production' == process.env.NODE_ENV
  var config = module.exports = {
      // the base path which will be used to resolve entry points
      context: __dirname,
      // the main entry point for our application's frontend JS
      entry: {
          application: "./app/assets/javascripts/application.js",
      },
      externals: {
          "jquery": "jQuery",
          "$": "jQuery"
      },
      // devtool: (isProduction ? "hidden" : "#inline-source-map"),
  };
  config.output = {
      // this is our app/assets/javascripts directory, which is part of the Sprockets pipeline
      path: path.join(__dirname, 'app', 'assets', 'javascripts'),
      // the filename of the compiled bundle, e.g. app/assets/javascripts/bundle.js
      filename: '[name].bundle.js',
      // if the webpack code-splitting feature is enabled, this is the path it'll use to download bundles
      publicPath: '/assets',
      devtoolModuleFilenameTemplate: '[resourcePath]',
      devtoolFallbackModuleFilenameTemplate: '[resourcePath]?[hash]',
  };
  config.resolve = {
      extensions: ['', '.js', '.jsx', '.coffee', '.cjsx'],
      modulesDirectories: [ 'node_modules', 'bower_components' ],
      alias: {
          cldr: 'cldrjs'
      }
  };
  config.plugins = [
      new webpack.ResolverPlugin([
          new webpack.ResolverPlugin.DirectoryDescriptionFilePlugin('.bower.json', ['main'])
      ]),
      new webpack.ProvidePlugin({
          $: 'jquery',
          jQuery: 'jquery',
          "window.jQuery": "jquery"
      }),
      new webpack.DefinePlugin({
          'process.env': {'NODE_ENV': JSON.stringify(process.env.NODE_ENV)}
      })
  ];
  config.module = {
      loaders: [
          { loader: "babel-loader", test: [/\.js$/, /\.jsx$/], exclude: [new RegExp("node_modules"), new RegExp("bower_components")], query: {stage: 0} },
          { loader: 'coffee-loader', test: /\.coffee$/,  },
          { loader: "transform?coffee-reactify", test: /\.cjsx$/ },
      ],
  };
{% endhighlight %}

I also enabled the ES7 features for babel because I find it easier to write React classes with it. You can read more about this here.
The final build is generated as app/javascripts/application.bundle.js. You have to ignore this file from git and import it in rails html home page.
In production.rb I added this line to precompile bundeled fine in production mode:

{% highlight ruby %}
  config.assets.precompile += %w( application.bundle.js )
{% endhighlight %}

In development mode run webpack watch to create the bundle and update it on every change:

{% highlight shell %}
  bin/rake assets:webpack:watch
{% endhighlight %}

Now the client side application is decoupled from Rails, only the final bundled file is imported in rails.
One thing I recommend to be able to let React router handle the routes and not Rails is to add this line in routes.rb:


{% highlight ruby %}
  get '*unmatched_route', to: 'home#index'
{% endhighlight %}

This will send all the request to same page and let react router handle the paths.
