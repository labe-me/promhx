# Promhx

[![Build Status](https://travis-ci.org/jdonaldson/promhx.png)]
(https://travis-ci.org/jdonaldson/promhx)

Promhx is a [promise](http://en.wikipedia.org/wiki/Futures_and_promises) and
[functional reactive programming](TODO) library for [Haxe](http://www.haxe.org).
The "promise" and "stream" variables contain values that are not immediately
available. However, you can specify callback functions that will trigger when
the values do become available.

A typical case is to specify a callback for a given promise once the value
becomes available:

```as3
promise.then(function(p1) trace("do something with promise's value"));
```

Alternatively, you can specify a callback on multiple promise instances using
the static method "when":

```as3
Promise.when(promise1, promise2).then(function(p1,p2) trace("do something with the promise values"));
```

Streams work more or less the same:

```as3
stream.then(function(s1) trace("do something with the stream's value"));
```

```as3
Stream.whenever(stream1, stream2).then(function(s1,s2) trace("do something with the stream values"));
```

The major difference between Promises and Streams is that Promises may only
resolve once, while Streams may resolve multiple times.  Promises are suitable
for initialization and asset loading, while Streams are a useful alternative to
managing events.

Promhx has a number of powerful features:

* Fully cross-platform for php, c#, c++, java, js (nodejs and browser js), neko,
  and flash.
* Very efficient code that ranks among the fastest promise libraries for js.
* Type safety without requiring excessive boilerplate.
* Staggered promise/stream updates occur once per event loop, preventing
  excessive blocking of io in single threaded contexts (e.g. js).
* Event loop callbacks are provided where they make sense (js through
  setTimeout/setImmediate, flash through setTimeout).  You can also provide
  your own event loop callback for other platforms. See the "Event Loop" section
  for more details.
* Run time errors are propagated to subsequent promise/streams, and can be
  managed where appropriate.

Promises have the following behavior:

* Promises can only be resolved once.
* It is only possible to cancel a promise by rejecting it, which triggers an
  error.

Streams have the following behavior:
* If a stream is updated more than once in a single loop, the updates will
  happen once per loop in subsequent loops.
* Promises will remember their resolved value, and any functions specified
  afterwards by "then()" will get their result synchronously.

```as3
// Declare a promised value
var p1 = new Promise<Int>();

// Simple: deliver promise when value is available. Stream works the same.
p1.then(function(x) trace("delivered " + x));

// Deliver multiple promises when they are all available.
// the "then" function must match the types and arity of the contained values
// from the arguments to "when".
var p2 = new Promise<Int>();
Promise.when(p1,p2).then(function(x,y)trace(x+y));


// Stream has its own "when" based method, called "whenever".  Note that
// the returned stream value will resolve whenver *any one* of the stream
// arguments changes.
var s1 = new Stream<Int>();
var s2 = new Stream<Int>();
Stream.whenever(s1,s2).then(function(x,y)trace(x+y));


// Stream.whenever can mix and match stream and promise arguments:
Stream.whenever(s1,p1).then(function(x,y)trace(x+y));

// The return value is another promise, so you can chain.
Promise.when(p1,p2).then(function(x,y) return x+y)
    .then(function(x) trace(x+1));

var p3 = new Promise<String>();

// The pipe method lets you manually specify a new Promise to chain
// to.  It can be pre-existing, or created by the method itself.  Stream
// works in a similar fashion.
Promise.when(p1,p2).then(function(x,y) return x+y)
    .pipe(function(x) return p3)
    .then(function(x) trace(x));


// You can easily catch errors by specifying a callback.
Promise.when(p1,p2).then(function(x,y) throw('an error'))
    .error(function(x) trace(x));

// Errors are propagated through the promise chain.
// You can rethrow errors to use Haxe's try/catch feature.
// Stream works the same here too.
Promise.when(p1,p2).then(function(x,y) {throw('an error'); return 'hi';})
    .then(function(x) return 'a value')
    .error(function(x) {
        try {
            throw(x); // rethrow the error value to do standard error handling
        } catch(e:String){
            trace('caught a string: ' + e);
        } catch(e:Dynamic){
            trace('caught something unknown:' + e);
        }
    });

// If no error callback is specified, the error is thrown.
// Uncomment this next line to cause an error!
//Promise.when(p1,p2).then(function(x,y) throw('an error'));

// Promises can go through various stages before finally resolving.  The
// following methods check the status.


// Check to see if a promise has been resolved.  This will return true as soon
// as resolve() returns.
trace(p1.isResolved());

// Check to see if a promise is in the process of fulfilling.
// In some cases promises are not completely resolved.  This can happen if
// the promise is delaying execution (on flash, js), or is updating other
// promises.
trace(p1.isFulfilling());

// Check to see if the promise has completed fulfilling its updates.
trace(p1.isFulfilled());

// Check to see if a promise has been rejected.  This can happen if
// the promise throws an error, or if the current promise is waiting
// on a promise that has thrown an error.
trace(p1.isRejected());

// finally, resolve the promise values, which will start the
// evaluation of all promises.
p1.resolve(1);
p2.resolve(2);
p3.resolve('hi');

// You can "resolve" a stream as well since they share a base class, but an
// alias called "update" is provided to make this a bit clearer:
s1.resolve(1);
s1.update(1);
s2.update(2);
```

# Event Loop Management

When a promise or stream resolves, it can trigger a large amount of activity,
including the resolution of other promises and streams.  For single
threaded contexts, this can block other operations that require timely
execution (e.g.  IO/interaction functionality).  Promhx staggers the
resolution of promises and streams so that only one promise/stream will be
resolved per event loop.  However, *all* updates for a single promise/stream
will be executed in a given loop in order to ensure that all updates have
a consistent value.  If blocking continues to be a problem, consider using
more promises and streams to break the update operation up across multiple
event loops.


# Promhx Http Class
Promhx provides a promise-based Http class that is very similar to the
haxe.Http class in the base haxe library.  Note that you cannot change the url
to re-send the same request to different target urls (as in the original
haxe.Http class).

```as3
   var h = new promhx.haxe.Http("somefile.txt");
   h.then(function(x){
      trace(x); // this will be the text content from somefile.txt
   });
   h.request(); // initialize request.
```
# EventTools
Promhx provides some tools for adapting existing event systems into Streams and
Promises. To do so, it is recommended to import the ```promhx.haxe.EventTools```
class via "using":

```as3
  using promhx.haxe.EventTools;
  [...]

  var click_stream = element.eventStream("click");
  // click_stream type is Stream<Dynamic>;
```

# JQueryTools
Promhx has some JQuery-specific tools, also intended to be used via "using".

```as3
   using js.promhx.JQueryTools;
   [...]

   var target_click_stream = new JQuery("#target").eventStream('click');
   // target_click_stream is now a Stream<JqEvent>.
```

# Macro do-notation
Promhx has the ability to "compose" promise and streams using classes in the
promhx "mdo" module, and the [monax](https://github.com/sledorze/monax) library.
These macro functions
can be used as follows:

```as3
   import promhx.mdo.StreamM;
   [...]
   var s1 = new Stream<Int>();
   var s2 = new Stream<Int>();
   var s3 = StreamM.dO({
         val1 <= s1;
         val2 <= s2;
         ret({val1: val1, val2: val2});
   });
   s3.then(function(x){
      trace(x.val1);
      trace(x.val2);
   });

```


