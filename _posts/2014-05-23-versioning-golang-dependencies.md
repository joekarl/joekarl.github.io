---
layout: post
title: Versioning Golang Dependencies
---

A big beef I had in the past with go was the apparent lack of tools to manage your 
go environment. Pretty much the experience out of the box was to install packages 
globally (well user globally) which makes it hard to share code, hard to automate, 
and generally makes maintenance a pain.

##The GOPATH  
The `go` command is the supplied way to fetch, build, andinstall packages and 
commands. It relies on an environment variable named `GOPATH` to know where to install
dependencies. Normally the `GOPATH` is set to your home folder. This will work but 
ends up being a problem as multiple projects will all store their dependencies there.

Luckily this has been solved for us. [GVP](https://github.com/pote/gvp) is a tool that
will create a workspace specific to our project that can be used to vendor our 
dependencies. 

{% highlight sh %}
//Create a .godeps folder as a workspace
$ gvp init

//Activate the workspace
//Modifies GOPATH, GOBIN and PATH to use the .godeps
$ source gvp in

//Deactivate the workspace
//Restores the previous GOPATH, GOBIN and PATH.
$ source gvp out
{% endhighlight %}

The .godeps folder will be where all depencies will be installed. This is very similar 
to vendoring in Ruby, or virtualenv in Python. You should NOT check the .godeps folder
into your source repository.

##Go Dependencies
Go depencies are really just copies of git repositories. Unfortunately, the `go get`
command will only pull the master branch of a git repository. This means that any 
dependency really should be checked in with your source code so that you don\'t end
up with different environments having different versions of dependencies.

This is what we\'re trying to avoid.

Fortunately, again, this has been solved for us. 

###GPM
One option is to use [GPM](https://github.com/pote/gpm). GPM is a command line package 
manager with ability to specify which commit you want from a dependency.

{% highlight sh %}
//Create a Godeps file
$ gpm init

//Setup your Godeps file
$ cat Godeps
github.com/nu7hatch/gotrail               v0.0.2
github.com/replicon/fast-archiver         v1.02
launchpad.net/gocheck                     r2013.03.03   # Bazaar repositories are supported
code.google.com/p/go.example/hello/...    ae081cd1d6cc  # And so are Mercurial ones

//install dependencies
$ gpm install
>> Getting package github.com/nu7hatch/gotrail
>> Getting package github.com/replicon/fast-archiver
>> Getting package launchpad.net/gocheck
>> Getting package code.google.com/p/go.example/hello/...
>> Setting github.com/nu7hatch/gotrail to version v0.0.2
>> Setting github.com/replicon/fast-archiver to version v1.02
>> Setting code.google.com/p/go.example/hello/... to version ae081cd1d6cc
>> Setting launchpad.net/gocheck to version r2013.03.03
>> All Done
{% endhighlight %}

###Godeps
Another similar option is to use [Godep](https://github.com/tools/godep). Again, 
similar to the functionality of GPM. Godep also has the ability to validate the
current version of go required for the package

{% highlight sh %}
//Create a Godeps file
$ ls .
Godeps    main.go

$ cat Godeps
{
    "ImportPath": "github.com/kr/hk",
    "GoVersion": "go1.1.2",
    "Deps": [
        {
            "ImportPath": "code.google.com/p/go-netrc/netrc",
            "Rev": "28676070ab99"
        },
        {
            "ImportPath": "github.com/kr/binarydist",
            "Rev": "3380ade90f8b0dfa3e363fd7d7e941fa857d0d13"
        }
    ]
}

//install dependencies
$ godep get

//run go within godep sandbox
$ godep go
{% endhighlight %}

##When to use these tools?
Both GPM and Godep eliminate the step of having to checkin copies of depencies, 
but this causes your packages to not be `go get`able anymore. So if you\'re 
writing a library, you really probably want to go the traditional route with 
regard to depencies. But if you\'re building a standalone app, then both GPM
and Godep are perfect for reducing the source complexity of your application.

GVP will pretty much help in any application or library development and can 
be used in conjuction (or without) both GPM and Godep. 