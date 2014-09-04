---
layout: post
title: "Debugging in Node.js Part 2: Untangling Asynchronous Events"
description: ""
category: Software
tags: [software,web development, node.js]
---
{% include JB/setup %}

In the [last post](http://fullstackforum.com/software/2014/08/28/debugging-in-nodejs-part-1-the-basics/) we looked at a basic set of debugging tools which probably will help with about 90% of the bugs you encounter.  For some of those tough other 10%, I’ll go over some more specialized tools and techniques that help with some of the more unique aspects of Node (read: asynchronous events).  While print statements and breakpoints (and stepping through the code) are helpful when you have an easily repeatable error, the asynchronous nature of Node can make many errors seem almost impossible to diagnose.  By default, an event handler will render an incredibly useless stack trace, leaving you to wonder at what point your application actually went off the rails.  Hopefully the use of some of these tools will keep you from breaking a couple of keyboards in frustration.

###Long Stack Traces

All errors thrown by the v8 engine have the [same basic stack trace api](https://code.google.com/p/v8/wiki/JavaScriptStackTraceApi)).  Modules such as the [Node stack-trace module](https://www.npmjs.org/package/stack-trace) help expose this api, but what can you really do with it?  The [long-stack-trace module](https://github.com/tlrobinson/long-stack-traces) (or [longjohns module](https://github.com/mattinsler/longjohn), a fork of long-stack-trace) is one answer.  Long stack traces go beyond the basic stack trace you get with vanilla Node events, and instead allow your stack traces to span asynchronous events.  An example:

{% highlight javascript linenos=table %}
/* GET home page. */
router.get('/', function(req, res) {
  function someAsyncHandler() {
     throw new Error('Oh no! Event Error!');
  }

  setTimeout(someAsyncHandler, 1000);

  res.render('index', { title: 'Express' });
});

{% endhighlight %}

We get the following trace:

    /blab/routes/index.js:7
      throw new Error('Oh no! Event Error!');
           ^
    Error: Oh no! Event Error!
        at someAsyncHandler [as _onTimeout] (/Users/danielhood/Dev/Workspace/blab/routes/index.js:7:12)
        at Timer.listOnTimeout [as ontimeout] (timers.js:112:15)

Not very helpful, is it?  That’s because when the error is thrown, we only print the trace of the current stack frame (since it gets reset by any asynchronous operation).  However, adding the longjohns or long-stack-trace module wraps these asynchronous functions in order to store the current stack at the instant of registration.  It then uses this information and concatenates stack frames when an error is actually thrown.  As a result, we get much better looking and more useful stack traces (in this case I'm using longjohns):

    /Users/danielhood/Dev/Workspace/blab/node_modules/longjohn/dist/longjohn.js:185
            throw e;
                  ^
    Error: Oh no! Event Error!
        at someAsyncHandler (/Users/danielhood/Dev/Workspace/blab/routes/index.js:8:12)
        at listOnTimeout (timers.js:112:15)
    ---------------------------------------------
        at router.get.res.render.title (/Users/danielhood/Dev/Workspace/blab/routes/index.js:11:3)
        at next_layer (/Users/danielhood/Dev/Workspace/blab/node_modules/express/lib/router/route.js:103:13)
        at Route.dispatch (/Users/danielhood/Dev/Workspace/blab/node_modules/express/lib/router/route.js:107:5)
        at /Users/danielhood/Dev/Workspace/blab/node_modules/express/lib/router/index.js:205:24
        at proto.process_params (/Users/danielhood/Dev/Workspace/blab/node_modules/express/lib/router/index.js:269:12)
        at next (/Users/danielhood/Dev/Workspace/blab/node_modules/express/lib/router/index.js:199:19)
        ....//stack trace continues...

Note that the line after the break is where we originally set the timeout function.  This way we can see how our asynchronous events propagated.  Remember though, this method should only be used for debugging.  There is a reason that the Node core doesn’t generate these Error objects constantly, and that is because they are pretty expensive.  If you used these modules in a production environment, expect to have some serious performance issues.

###Domains and the TryCatch Module

[Domains](http://nodejs.org/api/domain.html) are a part of the Node.js core (introduced in v0.8) that help developers deal with errors in their applications instead of letting them cause a crash.  Domains let you declare a single point of exit for any uncaught errors, or namespace different errors in ways that allow you to define only a few specific event handlers.  For example, if we modify our code above:

{% highlight javascript linenos=table %}
var domain = require('domain');
var newDomain = domain.create();

/*An error handler for this domain*/
newDomain.on('error', function(err) {
  console.log('Error handled!');
});
/* GET home page. */
router.get('/', function(req, res) {

  function someAsyncHandler() {
     throw new Error('Oh no! Event Error!');
  }

  /* Any error that bubbles up from this run function will be caught
  by our domain */
  newDomain.run(function() {
    setTimeout(someAsyncHandler, 1000);
  });

  res.render('index', { title: 'Express' });
});
{% endhighlight %}

Now instead of crashing our program, we just get he message "Error handled!" every second.  By using Domains we can:

- Simplify error handling
- Create single points of exit for our application
- Gracefully handle errors so that we don't upset our users

Domains can also be used to store contexts for particular sessions (i.e. creating a new domain per each http connection), or to intercept errors from different modules (instead of using a typical callback) so that all errors can be handled in the same place.  There are many good uses for Domains, so [check out some more examples](http://nodejs.org/api/domain.html).

If you want an easy abstraction over the Domains (or just want to use them for error handling), the [trycatch](https://www.npmjs.org/package/trycatch) module is a solid choice.  Like the longjohn or long-stack-trace module, trycatch allows us to print long stack traces after a thrown error.  Instead of wrapping every asynchronous call like the former modules, it wraps every try function inside of its own domain.  If an error is thrown, trycatch uses the Domain functionality demonstrated above to give us access to the full stack trace.  The trycatch module is incredibly easy to use (a single function with try and catch functions as arguments), so its a great drop-in tool for grabbing long stack traces.  For the sake of completeness, here is a quick example from our code above:

{% highlight javascript linenos=table %}
var trycatch = require('trycatch')
trycatch.configure({
    'long-stack-traces': true
})
/* GET home page. */
router.get('/', function(req, res) {

  function someAsyncHandler() {
     throw new Error('Oh no! Event Error!');
  }

  /* Any error that bubbles up from this run function will be caught
  by our domain */
  trycatch(function() {
    setTimeout(someAsyncHandler, 1000);
  }, function(err) {
    console.log("Error Handled!", err.stack);
  });
  res.render('index', { title: 'Express' });
});
{% endhighlight %}

This results in similar stack-trace to what we had before.  Note that the trycatch module also allows you to color-code your output.

###Zones

Zones are a [faily recent addition](http://strongloop.com/strongblog/announcing-zones-for-node-js/) to the Node.js ecosystem.  They allow the creation of execution contexts (similar to continuation-local-storage) with the addition of long stack traces, layered exception handling, and a [bunch of cool features](http://strongloop.com/strongblog/comparing-node-js-promises-trycatch-zone-js-angular/) you should check out.  I'm going to hold off on going in-depth into zones since they are relatively new and I'm unsure about how "Zone.js overrides all asynchronous functions in the browser with custom implementations," but they are definitely a feature to watch.

###The Future With The AsyncListener API

So far we have looked at modules that help us debug asynchronous events by either collecting stack frames (expensive) or by using Domains.  Several developers have expressed a need for a more generic API that isn't quite as bulky as Domains, and still performant.  More specifically, Domains:

1.  Can't be turned off.  That is, once you attach a domain, it will always catch errors (even if you module is included in some larger app).

2.  Can't easily be used in a highly modular app.  By their very nature, Domains aren't the most modular of tools.  Attaching modules to a piece of code that uses Domains is automatically going to keep any errors from bubbling up.  Thus using Domains in one part of an app means that anything that is built on top of it will be stuck using the same error-handling paradigm.  And, because of (1), there really isn't a way to mitigate this issue.

3.  Can't scale to large applications easily.  This is really a result of 1 and 2, but trying to introduce Domains in a large project with several developers is going to cause a problem.  They likely aren't going to appreciate you attaching a domain to some part of the app (without discussing it) and changing how errors are dealt with.

The AsyncListener API is an answer to these issues.  Essentially, the goal of this API is to allow users to, as the name suggests, attach a listener to any type of asynchronous operation.  This, in turn, allows the easy creation of modules like long-stack-trace and trycatch without the massive performance overhead of saving stack frames or the unwieldiness of using Domains.  This API isn't set to relase until Node v0.12 (and is still [in a bit of flux](https://github.com/joyent/node/pull/8110)), but there is a [working polyfill](https://www.npmjs.org/package/async-listener) if you want to try it out now.  Once this API becomes current, expect to see many more asynchronous tracing and logging modules in the Npm registry.  

###Continuous Local Storage

Before I dive into CLS there are two important points you should remember:

1.  CLS can be used for much more than debugging--it allows for easier attachment of data along your call chain (for example, instead of attaching a bunch of properties to the req/res objects).

2.  It is relatively new, and much of the underlying AsyncListener API is still in flux.  Currently, it uses the [polyfill](https://www.npmjs.org/package/async-listener).

That being said, [continuation-local-storage](https://www.npmjs.org/package/continuation-local-storage) can be a pretty interesting tool for debugging and logging issues.  Essentially, it allows us to attach meta-data to a particular chain of execution, including asynchronous calls.  We don't have to wrap our whole chain in a particular listener (like a domain), and we can use it as a place to store additional data.  If you picture a chain of callbacks/events as something like a thread (in a more traditional multi-threaded language), then CLS is conceptually similar to thread-local storage.

From the [docs](https://www.npmjs.org/package/continuation-local-storage):

>A simple rule of thumb is anywhere where you might have set a property on the request or response objects in an HTTP handler, you can (and should) now use continuation-local storage.

This works great in our example Express app.  In our outer app.js file we have:

{% highlight javascript linenos=table %}
/* Grab the CLS module and initialize our new context */
var cls = require('continuation-local-storage');
var session = cls.createNamespace('mainSession');

app.use(function(req,res,next){
    req.db = db;

    /* bind all of the event handlers related to the req/res objects
        to our current session. */
    session.bindEmitter(req);
    session.bindEmitter(res);

    session.run(function() {
      /* we can even set session variables */
      session.set('userName', 'testPerson');
      /* Continue executing.  All under our 'mainSession' context */
      session.run(next());
    });

});

{% endhighlight %}

And in our routes index.js file:

{% highlight javascript linenos=table %}
var cls = require('continuation-local-storage');
/* GET home page. */
router.get('/', function(req, res) {

  var session = cls.getNamespace('mainSession');
  /* get the user from our CLS session */
  console.log(session.get('userName')); //prints 'testPerson'

  res.render('index', { title: 'Express' });
});
{% endhighlight %}

So this is a great way to isolate individual execution contexts in our applications (at least it is better than overloading the req/res objects all the time), but how does this help with debugging our apps?  CLS is more of a tool that can be used to build other debugging modules.  Using it you can easily record both syncronous and asynchronous calls in order to print long stack traces, pass objects to your error constructors for more detailed messages, or just implement basic logging across critical or problematic code.  All of this comes with more flexibility and less of a performance impact than the implementation of Domains.

As the AsyncListener API matures, expect to see more tools appear that make debugging Node's asynchronous events easier.  Core components such as the [experimental tracing module](http://nodejs.org/docs/v0.11.13/api/tracing.html) will slowly improve the overall efficacy of Node debugging and performance tuning.  Until then, the tools mentioned here (and others like them) will be the extent of our debugging toolbox.

In part 3 of this series I'll attempt to go over some of Node's performance profiling and tracing tools, such as JStrace, Dtrace, et al.  I'll also look at analyzing coredumps with GDB/LLDB, or at least defer to some better articles on the subject.
