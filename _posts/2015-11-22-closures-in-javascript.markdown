---
layout: post
title:  "Closures in Javascript"
date:   2015-11-22 22:39:00 -0500
categories: javascript closure
---
When I was interviewing for my last job, one of the companies I interviewed with asked me to describe what closures were in javascript.  I wasn't really happy with the answer that I gave at the time.  I am aware that there are countless places that describe closures across the internet.  Mozilla has very good writeup [here][closures].  However, since one of the best ways to learn something is to explain or teach it, I wanted to write up a simple description myself.

You can find all of the code used in this post [here][source].

TL/DR: A closure is a function that has one or more variables bound to it.

In the following simple example `greetingFactory` returns a function that can say hello to the name passed into it.  Notice that the sayHi function accesses the greeting variable from outside of its scope.  This is possible due to lexical scoping in javascript. 

{% highlight javascript %}
var greetingFactory = function() {
  var greeting = 'Hello there'  
  var sayHi = function(name) {
    return greeting + ' ' + name;
  }
  return sayHi;
}
{% endhighlight %} 

When `greetingFactory` returns, the `greeting` variable has gone out of scope.  However, its value is bound to the `sayHi` function.  The `sayHi` function remembers the environment in which it was created.  The return value is then assigned to the `hiGreeter` function below.  `hiGreeter` is now a closure.  `hiGreeter` now references the `sayHi` function with the value of the `greeting` variable enclosed in it.

{% highlight javascript %}
var hiGreeter = greetingFactory();
var out = hiGreeter('John')
console.log(out);
// This prints: Hello there John
{% endhighlight %} 

To make this example (ever so slightly) more interesting, I can pull the `greeting` value out as a parameter to `greetingFactory`.  I can now make multiple different functions using the factory that use different greeting values.

{% highlight javascript %}
greetingFactory = function(greeting) { 
  var sayHi = function(name) {
    return greeting + ' ' + name;
  }
  return sayHi;
}

var saluteGreeter = greetingFactory('Salutations');
var ahoyGreeter = greetingFactory('Ahoy hoy');

var salute = saluteGreeter('Alley');
console.log(salute);
// This prints : Salutations Alley

var ahoy = ahoyGreeter('Matthew');
console.log(ahoy);
// This prints: Ahoy hoy Matthew
{% endhighlight %} 

Feel free to yell at me if I got something wrong using the email below.  Thanks for reading (I say to absolutely no-one).

[closures]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Closures
[source]: https://gist.github.com/jpschroeder/b2b71c3a3a9227e0f048