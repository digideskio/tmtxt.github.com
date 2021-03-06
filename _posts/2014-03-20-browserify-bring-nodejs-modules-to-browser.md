---
layout: post
title: "Browserify - Bring Nodejs modules to browsers"
description: ""
categories: [javascript]
tags: []
---


# 1. Browserify in the command line

[Browserify](http://browserify.org/) helps you modularize your client side
Javascript code using Nodejs `require` style. It also supports transforming
Nodejs modules on npm to browser compatible ones. You can install browserify
using npm

{% highlight console %}
$ npm install -g browserify
{% endhighlight %}

Writing code with browserify is pretty easy and familiar with Nodejs developers.
Just write your code in Nodejs require style and then let browserify handle the
rest for you. For example you have 2 files, `foo.js` and `main.js` with the
content like this

- foo.js

{% highlight js %}
module.exports = function() {
    // do something and return value here
    return 1;
};
{% endhighlight %}

- main.js

{% highlight js %}
// include foo.js here
var foo = require('./foo.js');

// you can also include nodejs/npm modules. this example includes d3-browserify installed
// using npm (npm install d3-browserify)
var d3 = require('d3-browserify');

// call the foo function
foo();  // returns 1

// call the d3
var tree = d3.layout.tree(); // create a tree layout in d3
{% endhighlight %}

<!-- more -->

There is nothing special here. The code is written in Nodejs style and is very
easy to understand for Nodejs developers. Of course, this cannot be executed in
browsers since they don't offer the `require` function. Now we will use
browserify to make it compatible with browsers environment. The easiest way is
to pack everything to a bundle with browserify

{% highlight console %}
$ browserify main.js > bundle.js
{% endhighlight %}

This command will bundle everything you need to run (foo.js, d3-browserify and main.js) to a
file named **bundle.js**. That file can be executed in browser. To use it,
simply include it using the `<script>` tag

{% highlight html %}
<script src="bundle.js"></script>
{% endhighlight %}

However, there is a problem with this command, that is it packs everything in a
single file. As a result, every output file will contain all the information
about required libraries that it needs. This makes the file size much bigger and
the browsers have to reload all those libraries each time it load the new file.
To take advantage of browser's cache functionality, you can use
**external requires** provided by browserify. Back to my previous example, now
we will transform foo.js, main.js and d3-browserify to 3 separate files. The
content of those 3 files remain unchanged.

- Expose foo.js -> foo-out.js and d3-browserify -> d3-browserify-out.js

{% highlight console %}
$ browserify -r foo.js > foo-out.js
$ browserify -r d3-browserify > d3-browserify-out.js
{% endhighlight %}

- For main.js

{% highlight console %}
$ browserify -x ./foo.js -x d3-browserify main.js > main-bundle.js
{% endhighlight %}

In the browser, you need to include all those 3 files

{% highlight html %}
<script src="foo-out.js"></script>
<script src="d3-browserify-out.js"></script>
<script src="main-bundle.js"></script>
{% endhighlight %}

Since you have expose foo.js to a module, in the browser you can also use
require

{% highlight html %}
<script language="javascript">
    var foo = require('./foo.js');
</script>
{% endhighlight %}

Later when you have another module which uses foo.js, for example **bar.js**,
browserify using -x for referencing to the old file

{% highlight console %}
$ browserify -x ./foo.js > bar-out.js
{% endhighlight %}

That's all the basic commands you need to know to work with browserify. The next
part will talk about automate browserify using Gulp.

# 2. Using Browserify with Gulp

If you haven't used Gulp before, you should take a look at this post for some
basic tasks
[Automate Javascript development with Gulp]({%post_url 2014-03-14-automate-javascript-development-with-gulp%}).

> **Update Jun 16 2014**: gulp-browserify lacks some of browserify features so I
> have updated the post to use browserify stream directly

Browserify can be used with Gulp since the object returned by Browserify is a
stream compatible with Gulp. However, you need
[vinyl-source-stream](https://www.npmjs.org/package/vinyl-source-stream) to
achieve this.

## 2.1 Browserify Node and npm packages

To browserify Node and npm packages, for example
[react](https://www.npmjs.org/package/react) on npm, use the browserify
APIs and pipe it to vinyl source and gulp dest like this

{% highlight js %}
var browserify = require('browserify');
var source = require("vinyl-source-stream");

// Browserify npm packages
gulp.task('browserify-lib', function(){
  // create a browserify bundle
  var b = browserify();

  // make react available from outside
  b.require('react');

  // start bundling
  b.bundle()
    .pipe(source('react.js'))   // the output file is react.js
    .pipe(gulp.dest('./dist')); // and is put into dist folder
});
{% endhighlight %}

When you run this task as `gulp browserify-lib`, it is equal to this command

{% highlight console %}
$ browserify -r react > ./dist/react.js
{% endhighlight %}

## 2.2 Browserify custom modules

It's similar when you want to browserify the module that you wrote

{% highlight js %}
// Browserify modules
gulp.task('browserify-modules', function(){
  // create a browserify bundle
  var b = browserify();

  // add the module file
  b.add('./main.js');

  // reference to the react module that we have browserified from previous step
  b.external(require.resolve('react', {expose: 'react'}));

  // make it available from outside with require('./main.js');
  b.require('./main.js');

  // start bundling
  b.bundle()
    .pipe(source('main.js'))    // the output file is main.js
    .pipe(gulp.dest('./dist')); // and is put into dist folder
});
{% endhighlight %}

When you run this task as `gulp browserify-modules`, it is equal to this command

{% highlight console %}
$ browserify -x react -r main.js > ./dist/main.js
{% endhighlight %}

Here is my sample
[gulpfile.js](/files/2014-03-20-browserify-bring-nodejs-modules-to-browser/paste.html)

## 2.3 Browserify client-side libraries

> **Update Jun 18 2014**: before that, I said that we shouldn't use browserify
> for client-side libraries. Now, I found the solution.

If the libraries you want to browserify are designed for client-side environment
(browsers), do not try to find them on npm and then browserify them to the
client. Because for many libraries like jquery, jquery-ui, twitter bootstrap,
they relies on global object to work properly. The solution is just load those
libraries through the script tag like the traditional way that we used to do
(can also be installed with a package manager like [bower](http://bower.io/)) and
use browserify with [literalify](https://github.com/pluma/literalify) transform
to pretend those libraries are actual CommonJS modules.

This is an example with Jquery, Twitter Bootstrap and ReactJS installed through
Bower

- The HTML file, just load all those libraries via script as you usually do

{% highlight html %}
<!DOCTYPE html>
<html>
  <body>
    <script src="/bower/jquery/dist/jquery.js"></script>
    <script src="/bower/bootstrap/dist/js/bootstrap.js"></script>
    <script src="/bower/react/react.js"></script>
    <!-- your bundled browserify module here -->
    <script src="/dist/bundle.js"></script>
  </body>
</html>
{% endhighlight %}

- main.js

{% highlight js %}
// this is not the jquery or react installed from npm, it is the global object
// from script tag and we will use literalify to make them require-able
var jquery = require('jquery');
var react = require('react');

// pretend they are CommonJS modules
react.renderComponent(myComponent(), document.getElementById('content'));
jquery('#content');
// actually, they are just window.React or window.$
{% endhighlight %}

- gulpfile.js

{% highlight js %}
var browserify = require('browserify');
var source = require("vinyl-source-stream");

gulp.task('browserify', function(){
  var b = browserify();
  b.transform(literalify.configure({
    // map module names with global objects
    'jquery': 'window.$',
    'react': 'window.React'
  }));
  b.add('./main.js');
  b.bundle()
    .pipe(source('bundle.js'))
    .pipe(gulp.dest('dist'));
});
{% endhighlight %}

## 2.4 Auto browserify when there are changes

> **Update Aug 06 2014**: for watching Browserify bundle files, do not use
> `gulp.watch`, instead, use Watchify for that task. You can read about it here
> [Using Watchify with Gulp for fast Browserify build]({%post_url 2014-08-06-using-watchify-with-gulp-for-fast-browserify-build%}).

## 2.5 Browserify when files is in sub folder

**Note**: if you are placing the your files under a sub folder of your app’s
root directory(of course usually you do), you need to set the basedir for
browserify, for instance

{% highlight js %}
var b = browserify({
  basedir: './'
});
{% endhighlight %}

Now you can run all those previous commands normally from our Nodejs app root directory.

## 2.6 Sample gulpfile.js

Here is my sample [gulpfile.js](/files/2014-03-20-browserify-bring-nodejs-modules-to-browser/gulp.html)
which combine all of the above things
