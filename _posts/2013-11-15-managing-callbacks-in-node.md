---
layout: post
title: Managing callbacks in Node.js using Async.js
---
One of the big strengths of Node is the ability to defer IO to the background and handle the results asynchronously in a callback. Unfortunately that means callbacks, lots and lots of callbacks...

##Why is this a problem?
You\'ve seen this code before (and probably written it to):

{% highlight javascript %}

function fn(callback) {
    var connection = mysql.getConnection({...});

    var handleError = function(err) {
        connection.end();
        callback(err);
    }

    connection.query(sql, function(err2, result) {
        if (err2) return handleError(err2);
        doSomethingAsyncWithResult(result, function(err3, anotherResult){
            if (err3) return handleError(err3);
            connection.update(sql, [anotherResult], function (err4, anotherAnotherResult){
                if (err4) return handleError(err4);
                connection.end();
                callback(null, anotherAnotherResult);
            });
        });
    });
}

{% endhighlight %}

As you can see, we\'ve got our classic Node boomerang. We\'ve bastardized our variable names, made some awful error handling code, and generally made this hard to read and maintain.

##What can we do?
Don\'t use callbacks of course!

##What!? How will we get anything done without callbacks?
Okay, so we have to use callbacks, but, we can clean up how we set them up, split our code into discrete functions, and standardize how we handle errors.

##Enter Async.js
Async.js is an awesome library that abstracts all of the nastiness of callbacks away (see more here https://github.com/caolan/async). One of the big things that async does for us is to setup consistant callbacks and error handling.

##Ok, so we need to use async. Let\'s see some code!
Async provides a wonderful abstraction for creating pipelines. Basically what it does is take the result of the previous function in the pipeline and apply it to the next function to be called, eventually calling a callback when all finished. It also handles errors in any step of the pipeline. 

{% highlight javascript %}
var async = require('async');

function fn(callback) {
    var connection = mysql.getConnection({...});
    async.waterfall([
        doQuery.bind(null, connection),
        doSomethingAsyncWithResult,
        doUpdate(null, connection)
    ], function (err, result) {
        connection.end();
        callback(err, result);
    });
}

function doQuery(connection, callback) {
    connection.query(sql, callback);
}

function doSomethingAsyncWithResult(result, callback) {
    ... something ...
    callback(null, anotherResult);
}

function doUpdate(connection, result, callback) {
    connection.update(sql, [result], callback);
}
{% endhighlight %}

Neat huh? Basically we\'ve isolated all of our individual steps (doQuery, doSomethingAsyncWithResult, doUpdate), we\'ve isolated our db connection code, and anyone can look at the code and see exactly what order things are being performed in. Yay async.

##Ok but how does that work again?
So the first thing that async does is specify that each callback used should have this definition: `function (err, ...results){}`. So if you call the callback with an error (i.e. `callback(new Error())`) at any point of the pipeline, our ending callback will immediately get called with that error. If you call the callback with no error (i.e. `callback(null, someResult, anotherResult)`) then the next function in the pipeline will be called (and so on until the final call back is all that is left to be called). 

<div class="well">
<h3>Quick aside - Partial functions</h3>

<h4>#So whats with the whole bind thing?</h4>
If you notice, both doQuery and doUpdate need our db connection. However, doQuery is the first task in our pipline and doUpdate needs the result of the previous task (that doesn\'t have access to the db connection). 
<br /><br />
So what we\'re doing is creating a partial function that gives the doQuery and doUpdate functions access to the db connection without having to supply it in the function call.
<br /><br />
<h4>That\'s magic I say! Magic isn\'t good for code!</h4>
Well let\'s look at exactly what\'s going on here by creating our own bind function (this will be contrived and only handle simple functions with 2 arguments.
<br /><br />
{% highlight javascript %}
function myBind(fn, arg1) {
    return function(arg2) {
        fn(arg1, arg2);
    };
}

//and we can use it like this
function add(a, b) { return a + b; }

add(1, 1); //2
add(1, 2); //3

var add2 = myBind(add, 2);
add2(1); //3
add2(2); //4

var add5 = myBind(add, 5);
add5(1); //6
add5(2); //7
{% endhighlight %}
<br /><br />
Basically we create a new function that can be called with a limited subset of arguments (note, this is contrived b/c it can only handle 2arity functions).
<br /><br />
This is what we\'re doing with the db connection with bind and our query and update functions. We setup a partial function that has db connection pre-bound into it so we don\'t have to pass it in from the previous pipeline task.
</div>

##Okay, what else can we do with async?
So async also provides a bunch of other stuff you can do, specifically we\'ll look at 2 other flow control mechanisms: async.series and async.parallel. As you might expect, async.series does it\'s tasks in series and async.parallel does it\'s tasks in parallel. On completion of all tasks, the final callback will be called with the results of each task returned in order in an array.

Let\'s see it in action:

###Series
{% highlight javascript %}
async.series([
    wait5SecondsAndReturn1,
    wait5SecondsAndReturn2
], function (err, results){
    //after 10 seconds, results will be [1, 2]
});

function wait5SecondsAndReturn1(callback) {
    setTimeout(function(){
        callback(null, 1);
    }, 5000);
}

function wait5SecondsAndReturn2(callback) {
    setTimeout(function(){
        callback(null, 2);
    }, 5000);
}
{% endhighlight %}

###Parallel
{% highlight javascript %}
async.parallel([
    wait5SecondsAndReturn1,
    wait5SecondsAndReturn2
], function (err, results){
    //after 5 seconds, results will be [1, 2]
});

function wait5SecondsAndReturn1(callback) {
    setTimeout(function(){
        callback(null, 1);
    }, 5000);
}

function wait5SecondsAndReturn2(callback) {
    setTimeout(function(){
        callback(null, 2);
    }, 5000);
}
{% endhighlight %}

Note that we didn\'t have to change our functions to switch from doing things in series or in parallel. Also note that again, our code is nice and clean and easy to decipher what the code is doing. Basically everything looks like it is working in a procedural fashion while avoiding any blocking calls.

##So what did we learn?
Basically managing callbacks on our sucks in node and async.js can help. Yay async.js. I suggest using it. Everywhere...