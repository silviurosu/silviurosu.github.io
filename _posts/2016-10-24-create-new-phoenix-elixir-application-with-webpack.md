---
layout: post
cover: 'assets/images/cover2.jpg'
title: Create new Phoenix (Elixir) application with webpack and React
description: Creating a new Phoenix application using webpack and React
date: 2016-10-24 10:00:00
tags: Architecture Elixir Phoenix React webpack
subclass: 'post tag-test tag-content'
categories: 'Elixir'
navigation: True
comments: true
---

Since React and webpack are a standard for me on any new single page app I creted this step by step guide on how to setup a new Phoenix app and replace brunch with webpack and React to create single page applications.


Fist stept is to create a new phoenix app:
{% highlight shell %}
  mix phoenix.new admin
{% endhighlight %}

Do not install dependencies when asked. We will install them later when we add webpack

Then enter this new directory and update all dependencies:
{% highlight shell %}
  mix deps.update
{% endhighlight %}

Delete *brunch-config.js* file since we do not need it anymore

Replace current *package.json* content with:

{% highlight json %}
  {
  "repository": {},
  "license": "MIT",
  "dependencies": {
    "autoprefixer": "^6.5.1",
    "babel-core": "^6.17.0",
    "babel-eslint": "^7.0.0",
    "babel-loader": "^6.2.5",
    "babel-plugin-transform-runtime": "^6.15.0",
    "babel-preset-es2015": "^6.16.0",
    "babel-preset-react": "^6.16.0",
    "babel-preset-stage-0": "^6.16.0",
    "baconjs": "^0.7.88",
    "bootstrap-sass": "^3.3.7",
    "chart.js": "^2.3.0",
    "classnames": "^2.2.5",
    "copy-webpack-plugin": "^3.0.1",
    "css-loader": "^0.25.0",
    "css-mqpacker": "^5.0.1",
    "extract-text-webpack-plugin": "^1.0.1",
    "file-loader": "^0.9.0",
    "moment": "^2.15.1",
    "node-sass": "^3.10.1",
    "postcss": "^5.2.4",
    "postcss-loader": "^1.0.0",
    "postcss-scss": "^0.3.1",
    "ramda": "^0.22.1",
    "react": "^15.3.2",
    "react-bootstrap": "^0.30.5",
    "react-bootstrap-daterangepicker": "^3.2.2",
    "react-chartjs": "^0.8.0",
    "react-dom": "^15.3.2",
    "react-notification-system": "^0.2.10",
    "react-router": "^2.8.1",
    "sass-loader": "^4.0.2",
    "style-loader": "^0.13.1",
    "url-loader": "^0.5.7",
    "webpack": "^1.13.2",
    "webpack-manifest-plugin": "^1.0.1"
  },
  "devDependencies": {
    "eslint": "^3.8.0",
    "eslint-config-airbnb": "^12.0.0",
    "eslint-plugin-react": "^6.4.1"
  },
  "scripts": {
    "compile": "webpack -p",
    "lint": "./node_modules/.bin/eslint priv/static/js/app.js"
  },
  "eslintConfig": {
    "env": {
      "es6": true
    }
  },
  "engines": {
    "node": "6.8.0",
    "npm": "~3.8.1"
  }
}

{% endhighlight %}

If node packages are out of date you can use [node-check-updates](https://github.com/tjunnone/npm-check-updates) to update them to latest version.

Install all node dependencies by running:

{% highlight shell %}
  npm install
{% endhighlight %}

Edit *mix.exs* file and update elixir to later version (current 1.3)

Edit *config/dev.exs* and replace this lines:

{% highlight elixir %}
  watchers: [node: ["node_modules/brunch/bin/brunch", "watch", "--stdin",
                    cd: Path.expand("../", __DIR__)]]
{% endhighlight %}
with
{% highlight elixir %}
  watchers: [node: ["node_modules/webpack/bin/webpack.js", "--watch-stdin", "--progress", "--colors",
             cd: Path.expand("../", __DIR__)]]
{% endhighlight %}


Create a new webpack.config.js and update it to your preferences. This are my settings:
{% highlight javascript %}
  var ExtractTextPlugin = require("extract-text-webpack-plugin");
  var CopyWebpackPlugin = require("copy-webpack-plugin");
  var ManifestPlugin = require('webpack-manifest-plugin');
  var webpack = require("webpack");
  var autoprefixer = require('autoprefixer');
  var precss       = require('postcss');
  var mqpacker =  require('css-mqpacker');
  var postcss_scss = require('postcss-scss');

  module.exports = {
    devtool: 'source-map',
    entry: ['./web/static/js/app.js'],
    output: {
      path: './priv/static',
      filename: 'js/app.js'
    },
    resolve: {
      extensions: ['', '.js', '.jsx'],
      modulesDirectories: [ 'node_modules', __dirname + "/web/static/js" ],
      alias: {
        phoenix_html: __dirname + "/deps/phoenix_html/web/static/js/phoenix_html.js",
        phoenix: __dirname + "/deps/phoenix/web/static/js/phoenix.js"
      }
    },
    module: {
      loaders: [
        {
          test: [/\.js$/, /\.jsx$/],
          loader: 'babel-loader',
          exclude: /node_modules/,
          query: {
            plugins: ['transform-runtime'],
            presets: ['es2015', 'stage-0', 'react']
          }
        }, {
          test: /\.css$/,
          loader: 'style-loader!css-loader!postcss-loader'
        }, {
          test: /\.scss$/,
          // loader: 'style-loader!css-loader!sass-loader!postcss-loader'
          loader: ExtractTextPlugin.extract(
            'style',
            'css!sass?includePaths[]=' + __dirname +  '/node_modules/bootstrap-sass/assets/stylesheets&includePaths[]=' + __dirname +  '/web/static/css',
            'postcss'
          )
        },
        { test: /\.woff(\?v=\d+\.\d+\.\d+)?$/,   loader: 'url?limit=10000&mimetype=application/font-woff&name=../fonts/[name].[ext]' },
        { test: /\.woff2(\?v=\d+\.\d+\.\d+)?$/,  loader: 'url?limit=10000&mimetype=application/font-woff&name=../fonts/[name].[ext]' },
        { test: /\.ttf(\?v=\d+\.\d+\.\d+)?$/,    loader: 'url?limit=10000&mimetype=application/octet-stream&name=../fonts/[name].[ext]' },
        { test: /\.eot(\?v=\d+\.\d+\.\d+)?$/,    loader: 'url?limit=10000&mimetype=application/octet-stream&name=../fonts/[name].[ext]' },
        { test: /\.svg(\?v=\d+\.\d+\.\d+)?$/,    loader: 'url?limit=10000&mimetype=image/svg+xml&name=../fonts/[name].[ext]' }
      ]
    },
    // postcss: function () {
    //   return [autoprefixer, precss, mqpacker];
    // },
    plugins: [
      new ExtractTextPlugin("css/app.css"),
      new webpack.optimize.CommonsChunkPlugin("js/common.bundle.js"),
      new CopyWebpackPlugin([
        { from: "./web/static/assets" }
      ]),
      new ManifestPlugin({
        fileName: 'manifest.json',
        basePath: '/priv/static'
      })
    ]
  };
{% endhighlight %}

We need to create also development database by running:

{% highlight shell %}
  mix ecto.create
{% endhighlight %}

Then we will edit the default layout and replace the current <body> tag with:

{% highlight html %}
  <body>
    <div id="react-container"></div>
    <script src="<%= static_path(@conn, "/js/common.bundle.js") %>"></script>
    <script src="<%= static_path(@conn, "/js/app.js") %>"></script>
  </body>
{% endhighlight %}

Then you can start to write your application login inside *app.js*. If you want to see a simple page setup for React with routes and a few components please check [my repository](https://github.com/silviurosu/phoenix-webpack-react-template) where I put all this code.

In the next tutorial I will explain how to configure and deploy Phoenix app to heroku.
