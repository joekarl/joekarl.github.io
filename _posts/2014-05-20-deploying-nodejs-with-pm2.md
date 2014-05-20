---
layout: post
title: Deploying Node.js with PM2
---

One of the more mysterious parts of software development has always been
deploying applications. It\'s always been fairly easy to get an app up and
running, but it\'s pretty hard to keep it up and running. And what do you do
with logging? Monitoring? Using your hardware efficiently?

##Node deployment out of the box
Out of the box Node.js deployment options are pretty barren. The best you can
look forward to are:

* Single threaded execution
* Uncaught exceptions will crash your process
* Only basic STDOUT/STDERR logging
* Non-existant monitoring

So what do we need?

* Efficient use of computing resources
* Uncaught exceptions should trigger an application restart
* Nice output logging
* Scriptable monitoring

##Solution 1 - The Node cluster module.
The cluster module is built into the Node core and provides part of the
functionality we\'re looking for. It gives us:

* Multi process implementation of clustering
* App restart on error
* Clean shutdown of child processes

Unfortunately, we have to write all of this code ourselves and on top of that,
it starts to invade our code with environment specific clustering code.

{% highlight javascript %}
// Include the cluster module
var cluster = require('cluster');

// Code to run if we're in the master process
if (cluster.isMaster) {

    ...

// Code to run if we're in a worker process
} else {

    ... //ie application setup

}
{% endhighlight %}

##Solution 2 - Forever Monitor
Forever Monitor is designed to work in conjuction with your code to provide
intelligent start/stop/restart handling. It gives us:

* Nice logging facilities
* App restart on error
* Clean shutdown of child processes

Sounds good, however in practice we still have to implement this handling in our
code. Also the clustering capabilities aren\'t that great.

{% highlight javascript %}
var forever = require('forever-monitor');

var child = new (forever.Monitor)('your-code.js', {
    max: 3,
    silent: true,
    options: {
        'logFile': 'path/to/file', // Path to log output from forever process (when daemonized)
        'outFile': 'path/to/file', // Path to log output from child stdout
        'errFile': 'path/to/file'  // Path to log output from child stderr
    }
});

child.on('exit', function () {
    console.log('your-code.js has exited after 3 restarts');
});

child.start();
{% endhighlight %}

##Solution 3 - PM2
PM2 (https://github.com/Unitech/pm2) is a library that tries to combine the best
parts of forever and the cluster module into a single package as well as
provide some nice monitoring features. It gives us:

* Built-in load balancer (using the native cluster module)
* Script daemonization
* 0s downtime reload for Node apps
* Generate SystemV/SystemD startup scripts (Ubuntu, Centos...)
* Pause unstable process (avoid infinite loop)
* Restart on file change with --watch
* Monitoring in console as well as a web endpoint
* Combined logging
* Coffeescript support

So what code changes do we need to make to get PM2 to work? Thats right! None!
Spectacular!

Basically PM2 wraps up all of the code that you\'d normally need to write to use
cluster and forever effectively without you having to write it.

####So what about the whole monitoring thing?
PM2 allows us to query the status of our apps by calling `pm2 jlist` or even
better we can run `pm2 web` to start a web endpoint which can be queried for
the same information. This makes it perfect for remote health checks of our
applications.

##What did we learn?
There is more than one way to deploy a Node.js application and none of them are
really wrong. However there\'s some tools we can use to make our lives easier.
PM2 is one of those tools. It provides a lot of the basic functionality that we
need to reliably run our apps in production while adding some essential parts
to really let our code be our focus without having to worry about how the app
is going to hold up in a failure scenario.
