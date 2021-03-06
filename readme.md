# Error Handling in Express

## Introduction

Q: What can go wrong in an Express app?
A: Everything

There are different kinds of errors:

* Software Bugs
  - infinite loop
  - unrecognized route
  - race condition
* System Failures
  - hard drive storage is full
  - network failure
  - database connection failure
  - out of memory error
* User Errors
  - user entered wrong login or password
  - user entered bad data (bad zip code, SSN, Credit Card #, etc.)

## Separation of Concerns

It is important to note that we often need to separate out the _detection_ of an error from the _handling_ of the error. One part of our code may detect the error and another part of our code may need to report on errors. How do we pass the error from the _detection_ code to the _error handler_?

First lets see how this is done as part of the JavaScript language.

## JavaScript Errors and Exception Handling

JavaScript provides the `throw`, `try`, and `catch` keywords for throwing, and handling exceptions. An _exception_ is an error that is being reported in a way that aborts the current flow of execution and unwinds the call stack looking for an _exception handler_ designated by the `try` and `catch` blocks. This is called "throwing an exception".

Let's look at an example:

ex1.js:

```javascript
function foo(data) {
  var sum = data.x() + data.y();
  console.log('sum =', sum);
}

function bar(data) {
  foo(data);
  console.log('we never get here');
}

try {
  bar('bad data');
}
catch(error) {
  var message = 'Oopsy, something went wrong:' + error;
  console.log(message);
  // throw new Error(message);
}

console.log('we do get here');
```

In the above example the JavaScript runtime engine detected an error while executing the 2nd line of the `foo` function. The JavaScript runtime then _threw_ an exception describing the error that occurred. We caught that exception in the `catch` block and printed the error to the console.

## Errors vs. Exceptions

In JavaScript (and Node.js especially), there's a difference between an _error_ and an _exception_. An _error_ is any instance of the `Error` class. Errors may be constructed and then passed directly to another function or thrown. When you `throw` an _error_, it becomes an _exception_

Here's an example of using an error as an exception:

```javascript
throw new Error('something bad happened');
```

but you can just as well use an Error without throwing it:

```javascript
callback(new Error('something bad happened'));
```

and this is much more common in Node because most errors are _asynchronous_.


## Error Handling in Express

Express uses _middleware_ to handle asynchronous requests and responses. Express gives us a way to plug into the middleware pipeline an error handler that will get called whenever an error is detected.




## Error Handler Middleware

If you weren’t aware of it, every _ExpressJS_ app comes with an error handler (actually two error handlers, one for development mode and one for production mode). These error handlers are included in the default `app.js` file that is generated by the `express` generator:

```javascript
// error handlers

// development error handler
// will print stacktrace
if (app.get('env') === 'development') {
  app.use(function(err, req, res, next) {
    res.status(err.status || 500);
    res.render('error', {
      message: err.message,
      error: err
    });
  });
}

// production error handler
// no stacktraces leaked to user
app.use(function(err, req, res, next) {
  res.status(err.status || 500);
  res.render('error', {
    message: err.message,
    error: {}
  });
});
```

This code properly handles an error that was sent up the line using the `return next(err);` style of handling. Instead of putting the app in to an exception state by throwing the error, it is properly handled by the middleware, allowing you to write your own custom code, error logging, and rendered view in response to the error ocurring.

> NOTE: You define error-handling middleware functions in the same way as other middleware functions, except for error-handling functions you have four arguments instead of three: (err, req, res, next).

> NOTE: You define error-handling middleware last, after other app.use() statements.


## Calling `next(err)` and `next('route')`

Calling `next(err)` tells the Express and Connect frameworks to pass the error along until an error handling middleware of function can properly take care of it.

```javascript
currentUser.save()
.then(function() {
  res.redirect('/todos');
}, function(err) {
  return next(err);         // pass the error to the next stage in the middleware pipeline
});
```

If you pass anything to the next() function (except the string 'route'), Express regards the current request as being in error and will skip any remaining non-error handling routing and middleware functions.

If you have a route handler with multiple callback functions you can use the route parameter to skip to the next route handler. For example:

```javascript
app.get('/a_paid_only_route',
  function checkIfPaidSubscriber(req, res, next) {
    if(!req.user.hasPaid) {
      next('route');          // skips the getPaidContent handler
    }
  }, function getPaidContent(req, res, next) {
    PaidContent.find(function(err, doc) {
      if(err) return next(err);
      res.json(doc);
    });
  });
```

## Running with DEBUG parameters

Compare the difference in terminal output between:

```bash
DEBUG=todos:* npm start
```

and

```bash
DEBUG=* npm start
```

> NOTE: The latter may be useful when debugging a really tricky problem.

How does this work?

Express uses the _debug_ module internally to log information about route matches, middleware functions that are in use, application mode, and the flow of the request-response cycle.

> `debug` is like an augmented version of `console.log`, but unlike `console.log`, you don’t have to comment out `debug` logs in production code. Logging is turned off by default and can be conditionally turned on by using the `DEBUG` environment variable.

To see all the internal logs used in Express, set the _DEBUG_ environment variable to express:* when launching your app.

```bash
DEBUG=express:* npm start
```

To see the logs only from the router implementation set the value of DEBUG to express:router

```bash
DEBUG=express:router npm start
```

You can combine DEBUG parameters using a comma-separated list of parameters:

```bash
DEBUG=http,mail,express:router,todos:* npm start
```

## Adding Your Own Debug Messages

With debug you simply invoke the exported function to generate your debug function, passing it a name which will determine if a noop function is returned, or a decorated console.error, so all of the console format string goodies you're used to work fine. A unique color is selected per-function for visibility.

For example, if we edit the `routes/todos.js` file and add the following `debug` code:

```javascript
var debug = require('debug')('todos:router');
...

// INDEX
router.get('/', authenticate, function(req, res, next) {
  debug('You found the INDEX route for todos!');
  var todos = global.currentUser.todos;
  res.render('todos/index', { todos: todos, message: req.flash() });
});
```

when we hit the `/todos` route, we will see the following in the Terminal output:

```
  todos:app You found the INDEX route for todos! +42s
```

> NOTE: The above output will only print if the DEBUG flag contains a matching pattern, for example: "todos:router" or "todos:*" or "*"


## Running in Development or Production Mode

```bash
npm start                     # run in development mode
NODE_ENV=production npm start # run in production mode
```

You can add the following line to the bottom of your `app.js` just above the line `module.exports = app;`:

```javascript
debug('Running in %s mode', app.get('env'));
```

Now when you start the app it will print out what mode you are currently using.

## Summary

* Express provides a `debug` module for defining named debug loggers:

```javascript
var debug = require('debug')('todos:app');
...
debug('Here is a debug message');
```

* We can enable and disable various `debug` loggers via the `DEBUG` variable:

```bash
DEBUG=express:router,todos:* npm start
```

* Most modern programming languages (including JavaScript) support the detection and reporting of errors via _exceptions_.
* Exceptions work well for synchronous error handling, but it does not work well when we are running asynchronously.
* For async error handling we need to use callbacks.
* Express provides a way of connecting callbacks into a _middleware_ pipeline.
* Express error handlers are simply middleware that are defined to take 4 arguments instead of 3: (err, req, res, next)
* We can pass an error down the _middleware_ pipeline by calling `next`:

```javascript
currentUser.save()
.then(function(saved) {
  res.redirect('/todos');
}, function(err) {
  return next(err);         // pass the error down the middleware pipeline
});
```

## For Further Reading

* [Proper Error Handling in ExpressJS](http://derickbailey.com/2014/09/06/proper-error-handling-in-expressjs-route-handlers/)
* [Error Handling in Node.js](https://www.joyent.com/developers/node/design/errors)
* [Express Error Handling](http://expressjs.com/en/guide/error-handling.html)
* [The debug module](https://www.npmjs.com/package/debug)
* [Error Handler Module](https://github.com/expressjs/errorhandler)
