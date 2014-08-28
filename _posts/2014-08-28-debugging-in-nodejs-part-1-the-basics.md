---
layout: post
title: "Debugging in Node.js Part 1: The Basics"
description: ""
category: Software
tags: [software,web development, node.js]
---
{% include JB/setup %}

The state of debugging node.js apps has always been a bit unfortunate, and [continues to be a point of pain](https://medium.com/code-adventures/farewell-node-js-4ba9e7f3e52b) for many.  While there is plenty of room for improvement, there is also a [sizeable ecosystem](https://www.npmjs.org/browse/keyword/debug) of tools that developers have available.  In this series, I'll try to touch on everything that I have used or find interesting.  I’m not sure how in depth to go, but I’ll start with some of the more basic tools and continue until I feel like I’m rambling.  If I miss something you find essential, feel free to contact me and I can add it in!

### Starting Tools (JSHint, Nodemon, and Pre-commit Hooks)
These tools fall more into the “preventative care” category of debugging.  While not mandatory by any means, they can save a ton of time in the long run.  

JSHint is one of several "linters" that will warn you about potential errors or stylistic inconsistancies in your Javascript code.  JSHint will warn you about a variety of potential errors, such as:

{% highlight javascript linenos=table %}
function someFunction (foo, bar) { //'bar' is defined but never used
    condition (fooi) { //'fooi' is undefined
          case 'one' :
              anotherFunction1() //will warn about the missing break
          case 'two' :
              anotherFunction2();
              break;
    }
    return foo  //will warn about a missing semicolon
}
{% endhighlight%}

There are tons of options and preferences you can set in addition to comment flags in the code (to disable warnings for certain blocks/functions).  I find the linter an invaluable tool for catching simple typos and oversights before they become an issue.  Make sure to [check out the docs](http://www.jshint.com/docs/) for all of the available options.

[Nodemon](http://nodemon.io/) is another helpful tool that will watch your files for changes and restart the server automatically instead of you having to manually restart the process.  There are a bunch of similar utilities to do the same  (i.e. grunt), but all of them help to automate a task you would otherwise repeat a few thousand times a month.  It isn't going to help with errors, but it might help you to not accidentally forget restarting the server after a change when you are burning the midnight oil.

The last awesome tool I’ll mention is [pre-commit hooks](https://www.npmjs.org/package/precommit-hook) for Git.  Using a commit-hook to automatically run your linters/tests with each code commit ensures you are never accidentally checking in bad code (well, not never, but potentially much less).  Automating tests/linting is infinitely better than running it manually, and it encourages a constant use of best practices.  I feel that this last tool is much too rarely used by many developers.

I’m not going go into any depth about testing in these posts.  It is quite simply too important a topic with too many potential options, and thus warrants a post of its own (well, probably a book).  Just know that automated testing of all types (smoke, unit, and functional tests) is essential to finding bugs early and limiting time spent debugging.

### Basic Console Logging

The simplest and probably most used tool in a developer's debugging repertoire are print statements.  Console.log, I’m sure, is used by most everyone--but there are several other useful functions in the console family/module.  The [console documentation](http://nodejs.org/api/stdio.html) is worth a read, but here are some quick examples of the different functions available:

{% highlight javascript linenos=table %}
/* start a timer for some quick benchmarking */
console.time('myTimerLabel');
var myObject = {
    foo : 'bar',
    something : 'else'
  }

/*Console.log passes its arguments through util.inspect, which prints to
stdout.  For example, below I use %s:%j to format my first argument as a
string, and my second as a JSON'ed object*/
console.log('%s:%j', 'myObject', myObject); //myObject:{"foo":"bar","something":"else”}

/*same as console.log, but printed to stderr */
console.error('oh no', myObject); //oh no { foo: 'bar', something: 'else' }

/* Calls util.inspect() on the supplied object */
console.dir(myObject); //{ foo: 'bar', something: 'else’ }

/*Ends our timer with the given label */
console.timeEnd('myTimerLabel'); //myTimerLabel: 1ms

/*simple assertion tests with label.  Prints nothing if true, otherwise
throws an error and prints a stack trace */
console.assert(myObject.foo === 'bar', 'test if myObject has property foo'); //{no output}

/*Prints a stack trace with the supplied label */
  console.trace('stack trace');  //Trace: stack trace ... {stack trace follows}
{% endhighlight %}

No rocket surgery here, but hopefully you can pick up a few of the more nuanced uses.

### Debuggers

There are several different helpful debugging programs and tools you can find in the Npm registry (and other places), but in regards to debuggers I’ll stick to three:  [the default node debugger](http://nodejs.org/api/debugger.html), [npm’s debugging package](http://nodejs.org/api/debugger.html), and the excellent [node inspector](https://github.com/node-inspector/node-inspector).  Two of these debuggers use the [v8 debugging protocol](https://code.google.com/p/v8/wiki/DebuggerProtocol), but they each give us varying levels of support.  

#####  Npm Debug Module

[The debug module](https://github.com/visionmedia/debug) is more of an enhancement to Node’s logging functions than a true debugger.  It doesn’t interface with v8’s profiler (as the next two tools do), but it is still a popular module that can add some useful features.  Once you require debug in your respective module, you can use the exported function to print log errors to stderr and stdout.  Debug will color code the log statements by function, and also allow you to monitor the time between calls.  A screenshot showing the type of output you can generate is below:

![Npm Debugger]({{site.url}}/assets/images/npmDebug.png "Npm Debugger")

Another benefit of this module is that it also integrates seamlessly with client-side code (and is backed by local-storage to save messages), so if you have a bug that is showing up on both client and server environments, you can use this module to easily compare output.

##### Default Node Debugger

The default Node debugger is essentially just a basic CLI wrapper for the v8 profiler.  Essentially it just supports setting breakpoints, stepping through those breakpoints, and placing watchers on variables.  Watchers are simple expressions that will evaluate specified variables whenever a breakpoint is triggered, and they are fairly easy to use.  Note most of these examples are going to be used with [a simple express app](http://cwbuecheler.com/web/tutorials/2013/node-express-mongo/ ) that I’ve altered a bit (so you can follow along/reproduce if you wish).  In this case, we have a simple route that receives form input, and adds a new user to a Mongo DB:

{% highlight javascript linenos=table %}
/* Add a user to our DB.  No validation/security for now... */
router.post('/adduser', function(req, res) {
    /* Our database object, and some form values to store in it. */
    var db = req.db,
      userName = req.body.username,
      userEmail = req.body.useremail,
      collection = db.get('usercollection');

    /* set a breakpoint here! */
    debugger;
    collection.insert({
        "username" : userName,
        "email" : userEmail
    }, function (err, doc) {
        if (err) {
            res.send("Oh no! An Error!");
        } else {
            /*redirect to userlist page*/
            res.location("userlist");
            res.redirect("userlist");
        }
    });
});
{% endhighlight %}
Starting the app with “node debug app.js”  will give us the following in the console:

    break in app.js:1
    1 var express = require('express');
    2 var path = require('path');
    3 var favicon = require('static-favicon');
    debug>

By default, the debugger will automatically break at the entry point to the application.  We can get a list of commands by typing help:

    debug> help
    Commands: run (r), cont (c), next (n), step (s), out (o), backtrace (bt), setBreakpoint (sb), clearBreakpoint (cb), watch, unwatch, watchers, repl, restart, kill, list, scripts, breakOnException, breakpoints, version

There’s nothing too crazy here.  We can set watchers on expressions or use a repl to evaluate different variables and perform basic operations in the console.  Anyway, after continuing from the opening break and attempting to insert a new user (Iama Testperson with an email of test@fake.net), we are met with this:

    break in /index.js:34
    32
    33     /* set a breakpoint here! */
    34     debugger;
    35     collection.insert({
    36         "username" : userName,
    debug>

From here we can evaluate all the variables in the local scope, print a trace, etc.  For example, if we were having issues inserting users into the database, we could check what those insertion values are:

    break in /index.js:34
    32
    33     /* set a breakpoint here! */
    34     debugger;
    35     collection.insert({
    36         "username" : userName,
    debug> repl
    Press Ctrl + C to leave debug repl
    > userName
    'Iama Testperson'
    > userEmail
    'test@fake.net'
    > userName + ', ' +  userEmail
    'Iama Testperson, test@fake.net'

It isn't the quickest to use, but it is pre-packaged with Node and can be useful when you need to do more than simple print statements.  Note that you can also debug an active process with the “SIGUSR1” command (which sets the process in debug mode) and then attaching the debugger CLI tool with: "node debug -p <pid>” or "node debug <URI>”.  The step-by-step for putting an active process into debug mode is farther below.

##### Node Inspector
The last debugger I want to mention is one of the most useful. If you are working on a large Node application, or are just a web developer in general, you more than likely have some experience with the awesome [Chrome developer tools](https://developer.chrome.com/devtools).  Node Inspector uses the same v8 protocol as the default Node debugger, but instead of a CLI with which to interact it uses a version of the Chrome developer tools interface.  This way you don't even have to learn anything new to debug Node apps! Sorta...

Node Inspector is easy enough to install:

    npm install -g node-inspector

Then just start Node Inspector with your app of choice:

    node-debug app.js

Once running, the inspector will open in your default browser (or you can point a browser at the address printed in the console).  Just like the Chrome debugger, Node Inspector allows you to:

- Navigate your app’s file structure and set breakpoints as it is running (or before a module is loaded).
- Perform all of the breakpoint operations as the standard Node debugger (step in, out, etc), all via a friendly GUI.
- Use the console as a REPL to evaluate variables and perform basic operations.
- Edit/hotswap running code and change variable values.
- Find information about variables, either with a mouseover or in the detailed contextual views of the current scope.

Quick Example:

![Node Inspector in all of its glory]({{site.url}}/assets/images/nodeInspector.png "Node Inspector in all of its glory")

Hitting a breakpoint, we see our code we can modify in the center, the local scope and call stack on the right, and our app directory on the left.  We can also hover over variables to quickly see their values (and modify them by double clicking).

You can also start the debugger on a currently running app.  This method is useful if you don’t want to restart your current process for some reason, i.e it has gotten into a strange state you aren’t sure how to reproduce or you are debugging a production app.  On my Mac it’s fairly simple:

    daniels-mbp:blab danielhood$ pgrep -l node //find the PID of the Node process
    51483 node
    daniels-mbp:blab danielhood$  kill -s USR1 51483  //send the USR1 signal to that process
    daniels-mbp:blab danielhood$

And on the terminal screen where the Node process is running, we get:

    Hit SIGUSR1 - starting debugger agent.
    debugger listening on port 5858

Now we can attach Node Inspector (with the command "node-inspector”) since we have just activated the v8 debugger that it plugs into.  Navigate to http://127.0.0.1:8080/debug?port=5858 and you are in!

I know I said I wouldn't go into testing, but a feature of Node Inspector I have to mention is the ability to use it on [Mocha tests](https://github.com/visionmedia/mocha).  Debugging unit tests after a big refactor can sometimes cause some tough issues, but Node Inspector allows us jump into each test and figure out why things aren't working the way they should.  To use Node Inspector with Mocha, just use the following command (once Mocha is installed):

    node-debug _mocha {test dir}

Note that we run it with "_mocha" and not just "mocha".  This switch is due to the fact that the regular mocha command is actually a tiny wrapper that executes "_mocha", so the debugger wouldn't be connected to the correct process.

Node Inspector is by far my preferred tool as far as debuggers go.  It is a bit heavier and may not be the first thing you turn to when you run into a bug (and it can take a while to start), but there aren’t many better options for getting a good picture of what is going on in your application (at least with an intuitive interface).  It has a few other features I haven’t mentioned, so I encourage you to, again, [read the docs](https://github.com/node-inspector/node-inspector).

I think that is quite enough for the first part of this series.  Next, I'll look at some more specialized tools that are helpful for debugging asynchronous operations, profiling long running applications, and analyzing core dumps.
