---
layout: post
title: Flow control libraries and performance (NodeJS)
---

When it comes to NodeJS (as with any language), there\'s a lot of different way
to do things. One major issue that people tend to run across in Node is how to
handle callbacks in a sane manner. There are a few options, with the major ones
being:

* Native callbacks
* Async.js
* One of the many A+ Promise libraries

Let\'s look at a few of these and in particular discuss their performance
characteristics.

##Native callbacks
This is how the Node core works. Most core functions return something similar
the following:

{% highlight javascript %}
var fs = require('fs');
fs.readFile('/etc/passwd', function(err, data){
    //do something with error or data
});
{% endhighlight %}

This is fairly straight forward but leaves us with a nagging problem, callback
hell.

Take for example needing to read multiple files:
{% highlight javascript %}
var fs = require('fs');
fs.readFile('/path/to/file1', function(err, data){
    //do something with error or data
    fs.readFile('/path/to/file2', function(err, data){
        //do something with error or data
        fs.readFile('/path/to/file3', function(err, data){
            //do something with error or data
        });
    });
});
{% endhighlight %}

As you can see, this is bad. This code is un-maintainable and doesn\'t even do
the file reads in parallel. This style of code is what flow control libraries
were designed to fix. Let\'s look at a few of them.

##Flow Control
The idea of flow control libraries is to provide a nice abstraction for the
developer to treat callback heavy code as procedural code. These flow control
libraries tend to fall into two categories: Promise/A+ style and callback style.

###Promise/A+ libraries
The A+ spec (https://github.com/promises-aplus/promises-spec) is a specification
to provide an interoperable abstraction for dealing with asynchronous operations
in Javascript. Having those abstractions can make implementations of callback
heavy code fairly simple and read very close to procedural code. This is a very
good thing when delegating work across a team as it makes the code much more
approachable.
As one can imagine, there are quite a few implementations of the
A+ spec (the whole list is here: https://github.com/promises-aplus/promises-spec/blob/master/implementations.md)
with the major implementations being:

* Q
* RSVP
* when.js

Almost all of the implementations look something like the following (this
example using Q specifically):

{% highlight javascript %}
var Promise = require('q');

function asyncCall() {
    return new Promise.Promise(function(resolve, reject, notifify){
        //doing something async like...
        resolve("some result");
    });
}

var aPromise = asyncCall();
aPromise.then(function(result){
    //do something with result
});

{% endhighlight %}

Internally each promise library is able to implement the internals differently
as only the outer spec needs to be respected. As expected, these implementation
differences lead to some wildly different performance characteristics. Let\'s
take a look at some of them.

####Q
One of the early implementations of promises. However age is not always a good
thing. Internally Q uses Object freezing to guarantee integrity of the promise.
This causes V8 to incur a large performance hit. The following code demonstrates
this performance hit:

{% highlight javascript %}
var Q = require('q');
var assert = require('assert');
var config = require('./config');

var runCount = config.runCount;

console.log(new Date().getTime());

for(var i = 0; i < runCount; ++i) {
    doRun(i, areWeDone)
}

function areWeDone(i) {
    if (i == runCount - 1) {
        console.log(new Date().getTime());
        process.exit(0);
    }
};

function doRun(i, cb) {
    var start = new Date().getTime();
    var promises = [wait10(), wait10(), wait10(), wait10()];
    Q.all(promises).then(function(){
        assert.ok(new Date().getTime() - start >= 10);
        cb(i);
    });
}

function wait10() {
    return new Q.Promise(function(resolve){
        setTimeout(resolve, 10);
    });
}

{% endhighlight %}

This code takes a long long time to return.

####RSVP
RSVP was designed to specifically be a lighter implementation of A+ Promises and
certainly lives up to that billing. With the same implementation code, we get a
much better result:

{% highlight javascript %}
var RSVP = require('rsvp');
var assert = require('assert');
var config = require('./config');

var runCount = config.runCount;

console.log(new Date().getTime());

for(var i = 0; i < runCount; ++i) {
    doRun(i, areWeDone)
}

function areWeDone(i) {
    if (i == runCount - 1) {
        console.log(new Date().getTime());
        process.exit(0);
    }
};

function doRun(i, cb) {
    var start = new Date().getTime();
    var promises = [wait10(), wait10(), wait10(), wait10()];
    RSVP.all(promises).then(function(){
        assert.ok(new Date().getTime() - start >= 10);
        cb(i);
    });
}

function wait10() {
    return new RSVP.Promise(function(resolve){
        setTimeout(resolve, 10);
    });
}

{% endhighlight %}

This code, which is the same as our other test with Q, runs in about 8 seconds.
Much quicker than the Q implementation with the same code.

####when.js
When.js was designed to be very performance driven while continuing to be A+
compliant.

{% highlight javascript %}
var when = require('when');
var assert = require('assert');
var config = require('./config');

var runCount = config.runCount;

console.log(new Date().getTime());

for(var i = 0; i < runCount; ++i) {
    doRun(i, areWeDone)
}

function areWeDone(i) {
    if (i == runCount - 1) {
        console.log(new Date().getTime());
        process.exit(0);
    }
};

function doRun(i, cb) {
    var start = new Date().getTime();
    var promises = [wait10(), wait10(), wait10(), wait10()];
    when.all(promises).then(function(){
        assert.ok(new Date().getTime() - start >= 10);
        cb(i);
    });
}

function wait10() {
    return new when.Promise(function(resolve){
        setTimeout(resolve, 10);
    });
}

{% endhighlight %}

When.js really shines here with an even faster time than RSVP clocking in around
4 seconds.

###Callback style libraries
When it comes to callback style libraries, there really is one major library
that everybody uses: async.js, though it is easy enough to roll your own that
we\'ll show an example of that as well. Compared to the promise implentations thes are
at a decidedly lower level. This has good and bad consequences, good in you have
over ordering (among other things) and bad in that you have to think about how
an implementation performs it\'s tasks. Also to be taken into consideration
is the impact of performance on this lower level of abstraction.

####Async.js
This library precedes the promise libraries by a bit and is centered around
compatibility with the node core. It gives quite a few implementations that
wrap some of the most used things one might need implement with callback based
code.

{% highlight javascript %}
var async = require('async');
var assert = require('assert');
var config = require('./config');

var runCount = config.runCount;

console.log(new Date().getTime());

for(var i = 0; i < runCount; ++i) {
    doRun(i, areWeDone)
}

function areWeDone(i) {
    if (i == runCount - 1) {
        console.log(new Date().getTime());
        process.exit(0);
    }
}

function doRun(i, cb) {
    var start = new Date().getTime();
    async.parallel([
      wait10,
      wait10,
      wait10,
      wait10
    ], function(){
        assert.ok(new Date().getTime() - start >= 10);
        cb(i);
    });
}

function wait10(cb) {
    setTimeout(cb, 10);
}

{% endhighlight %}

Here we\'re doing essentially the same things we were doing with the promise
libraries, but as we can see, we\'re passing around raw functions. In the
performance category, it performs pretty well at around 7 seconds.

####Native Callbacks (reprise)
While async.js provides some nice abstractions, it is still doing a lot to our
code. We can do better if we don\'t need all of async\'s features. When we
originally encounted native callbacks, we decided that the problem with them
was the lack of flow control. Let\'s write a little bit of code to alleviate
that issue while doing as little work as possible within the flow control code.

{% highlight javascript %}
var assert = require('assert');
var config = require('./config');

var runCount = config.runCount;

console.log(new Date().getTime());

for(var i = 0; i < runCount; ++i) {
    doRun(i, areWeDone)
}

function areWeDone(i) {
    if (i == runCount - 1) {
        console.log(new Date().getTime());
        process.exit(0);
    }
};

function doRun(i, cb) {
    var start = new Date().getTime();
    simpleAsync.parallelize([
      wait10,
      wait10,
      wait10,
      wait10
    ], function(errors, results){
        assert.equal(results.length, 4);
        assert.ok(new Date().getTime() - start >= 10);
        cb(i);
    });
}

function wait10(cb) {
    setTimeout(cb, 10);
}

/**
 * Simple parallelize function
 * Assumes fns array has function(err, callback) items and no nulls
 * This is naive, but no error checking == fast
 *
 * cb should be a function(errors[], results[])
 * where errors and results will be arrays containing the results of the fn call
 *
 * so if fn[0] returns 'x' and fn[1] returns an error,
 * then errors == [undefined, err_from_fn1] and results == ['x', undefined]
 */
function parallelize(fns, cb) {
    var fnsLength = fns.length,
        i = 0,
        countingCallback = makeCountNCallback(fnsLength, cb);
    for (i; i < fnsLength; ++i) {
        fns[i](makeIndexedCallback(i, countingCallback));
    }
}

/**
 * Simple serialize function
 * Assumes fns array has function(err, callback) items and no nulls
 * This is naive, but no error checking == fast
 *
 * cb should be a function(errors[], results[])
 * where errors and results will be arrays containing the results of the fn call
 *
 * so if fn[0] returns 'x' and fn[1] returns an error,
 * then errors == [undefined, err_from_fn1] and results == ['x', undefined]
 */
function serialize(fns, cb) {
    var fnsLength = fns.length,
        i = 0,
        errors = [],
        results = [];
    fns[i](makeChainedCallback(i, fns, errors, results, cb));
}

/**
 * Create a function that will call the next function in a chain
 * when finished
 */
function makeChainedCallback(i, fns, errors, results, cb) {
    return function(err, result) {
        if (err) errors[i] = err;
        results[i] = result;
        if (fns[i + 1]) {
            return fns[i + 1](makeChainedCallback(i + 1, fns, errors, results, cb));
        } else {
            return cb(errors, results);
        }
    }
}

/**
 * Create a function that will call a callback after n function calls
 */
function makeCountNCallback(n, cb) {
    var count = 0,
        results = [],
        errors = [];
    return function(index, err, result) {
        results[index] = result;
        if (err) errors[index] = err;
        if (++count == n) {
            cb(errors, results);
        }
    }
}

/**
 * Create a function that will call a callback with a specified index
 */
function makeIndexedCallback(i, cb) {
    return function(err, result) {
        cb(i, err, result);
    }
}

{% endhighlight %}

This is about as low level as it gets without callback hell. We\'ve provided
just a couple of flow control constructs (parallelize and serialize) to make
our native callback code just a bit nicer. As one can guess, this is a very
simple implementation that does very little. However, it is very powerful speed
wise. While all of the other implementations we\'ve looked out resorted to busy
waiting to resolve whether the tasks had completed or not, this does not. It
manually counts the number of results that have been resolved and when that
number matches the number we expect, it returns. This is highly primitive, but
is very fast. On the performance end this runs in around 0.8 seconds.

##What did we learn?
Basically, promises and other flow control libraries adds a really nice abstraction
to add to a development team\'s toolbox. However, that abstraction comes at a
cost to performance. But in 99.9% of situations, that performance cost will
be mitigated by the cost of doing IO. So ultimately, don\'t shy away from using
promises or async (unless it\'s Q, it\'s slow) due to perceived performance
gains or losses. If you think you\'re hitting a bottleneck due to a flow control
library, you probably aren\'t. Benchmark your code to identify where your code
and identify where your time is spent, then if it really is your code flow
library, then dive lower.
