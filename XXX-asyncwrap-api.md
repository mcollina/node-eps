| Title  | AsyncWrap API |
|--------|---------------|
| Author | @trevnorris   |
| Status | DRAFT         |
| Date   | 2016-04-05    |

## Description

Since its initial introduction along side the `AsyncListener` API, `AsyncWrap`
has slowly evolved to ensure a generalized API that would serve as a solid base
for module authors who wished to add listeners to the event loop's life cycle.
Some of the use cases `AsyncWrap` has covered are long stack traces,
continuation local storage, profiling of asynchronous requests and resource
tracking.

It must be clarified that the `AsyncWrap` API is not meant to abstract away
Node's implementation details, and is in fact intentional. Observing how
operations are performed, not simply how they appear to perform from the public
API, is a key point in its usability. For a real world example, a user was
having difficulty figuring out why their `http.get()` calls were taking so
long. Because `AsyncHooks` exposed the `dns.lookup()` request to resolve the
host being made by the `net` module it was trivial to write a performance
measurement that exposed the hidden culprit. Which was that DNS resolution was
taking much longer than expected.

A small amount of abstraction is necessary to accommodate the conceptual nature
of the API. For example, the `HTTPParser` is not destructed but instead placed
into an unused pool to be used later. Even so, at the time it is placed into
the pool  the `destroy()` callback will run and then the id assigned to that
instance will be removed. If that resource is again requested then it will be
assigned a new id and will run `init()` again.


## Goals

The intent for the initial release is to provide the most minimal set of API
hooks that don't inhibit module authors from writing tools addressing anything
in this problem space. In order to remain minimal, all potential features for
initial release will first be judged on whether they can be achieved by the
existing public API. If so then such a feature won't be included.

If the feature cannot be done with the existing public API then discussion will
be had on whether requested functionality should be included in the initial
API. If so then the best course of action to include the capabilities of the
request will be discussed and performed. Meaning, the resulting API may not be
the same as initially requested, but the user will be able to achieve the same
end result.

Performance impact of `AsyncWrap` should be zero if not being used, and near
zero while being used. The performance overhead of `AsyncHook` callbacks
supplied by the user should account for essentially all the performance
overhead.


## Terminology

Because of the potential ambiguity for those not familiar with the terms
"handle" and "request" they should be well defined.

Handles are a reference to a system resource. The resource handle can be a
simple identifier, such as a file descriptor, or it can be a pointer that
allows access to further information, such as `uv_tcp_t`. Realize that the
latter still relies on a file descriptor, but has been encapsulated in a data
structure containing additional information that allows the application to work
with it.

Requests are short lived data structures created to accomplish one task. The
callback for a request should always and only ever fire one time. When the
assigned task has either completed or encountered an error. Requests are used
by handles to perform work. Such as accepting a new connection or writing data
to disk.


## API

### Overview

```js
// Standard way of requiring a module. Snake case follows core module
// convention.
const async_wrap = require('async_wrap');

// List of asynchronous subsystems (e.g. CRYPTO or NET). Can be used for trace
// filtering of events.
const subsystems = async_wrap.subsystems;

// Return the list of classes that can call init(). For example, `TCPWRAP`.
const classes = async_wrap.classes;

// Pass in the class string passed to init() to retrieve the subsystem that
// class is associated with.
const subsystem = async_wrap.getSubsystemOf(class);

// Return the id of the current parent. Useful for tracking state and
// retrieving the handle of the current parent without needing to use an
// AsyncHook().
const id = async_wrap.currentParentId();

// Retrieve a handle by its id. The id will have been passed to the init()
// callback. Reason for this API is to prevent users from holding references
// to the handles themselves. Thus possibly preventing GC from cleaning up
// the handle and ever firing the destroy() callback.
const handle = async_wrap.getHandleById(n);

// XXX: API that allows you to get the API from an existing handle. This will be
// necessary to retrieve the id for a given handle for APIs like
// server.on('connection'. Investigate how plausible this is, and if it would
// be useful. Possibly have it be a getter on the FunctionTemplate.
const id = async_wrap.getHandleId(handle);

// init() is called during object construction. The handle will have not
// completed construction when this callback runs. So not all fields will have
// populated.
function init(id, type, parentId) { }

// before() is called just before the handle's callback is called. It can be
// called 0-N times for handles (e.g. TCPWRAP), and should be called exactly 1
// time for requests (e.g. FSREQWRAP).
// XXX: Why shouldn't the before callback be passed the callback that's about to
// be executed. Or at least some other data about the callback.

function before(id) { }

// after() is called just after the handle's callback has finished, and will be
// called regardless whether the handle's callback threw. If the handle's
// callback did throw then hasThrown will be true.
function after(id, hasThrown) { }

// destroy() is called when an AsyncWrap instance is destroyed. In cases like
// HTTPPARSER where the resource is reused, or timers where the handle is only
// a JS object, destroy() will be triggered manually soon after after() has
// completed.
function destroy(id) { }

// Create a new instance and add each callback to the corresponding hook.
var asyncHook = new AsyncHook({ init, before, after, destroy });

// Enable hooks for the current synchronous execution scope. This will ensure
// the hooks are not in effect in case of multiple returns, or if an exception
// is thrown.
asyncHook.scope();

// Allow callbacks of this AsyncHook instance to fire. This is not an implicit
// action after running the constructor, and must be explicitly run to begin
// executing callbacks.
asyncHook.enable();

// Disable listening for new asynchronous events. Though this will not prevent
// callbacks from firing on asynchronous chains that have already run within
// the scope of an enabled AsyncHook instance.
asyncHook.disable();

// Unlike disable(), prevent any hook callback from firing again in the future.
// Future hooks can be added by running scope()/enable() again.
// TODO: I am not sure all of this is possible. Investigation is needed.
asyncHook.remove();
```


### `async_wrap`

The object returned from `require('async_wrap')`.


#### `async_wrap.subsystems`

List of all subsystems that may trigger the `init` callback. Some of these
include `NET`, `CRYPTO` or `TIMER`. Each subsystem relates to one or more
classes. One thing these are useful for is quick filtering of `init()` calls
and determining if information should be logged, etc.

#### `async_wrap.classes`

List of classes that can trigger `init()`. These are the names of the classes
instantiated to perform a given asynchronous task. Some of these include
`TCPWRAP`, `PIPEWRAP` or `HTTPPARSER`. One thing these are useful for is to
know how to transverse the handle for inspection.

#### `async_wrap.getSubsystemOf(class)`

Returns the subsystem of the `class`. The `init()` callback passes the `class`
of the caller. To retrieve the subsystem of the `class` use `getSubsystemOf()`.

#### `async_wrap.getHandleById(id)`

Some handles are cleaned up by the garbage collector. Preventing the user from
being able to store those handles for future reference. Instead node keeps
internal references to each handle that aren't affected by GC. They can then be
retrieved via the `id` passed to all `AsyncHook` callbacks.

Retrieval of each handle during the `init` or `destroy` callback will return
an incomplete object, since some of the properties will have been removed. It
should, however, be safe to inspect at any time. To view a complete handle as
soon as possible it is recommended to place the code in a `nextTick()`:

```js
function init(id) {
  process.nextTick(() => {
    // Print out the completed handle.
    console.log(async_wrap.getHandleById(id));
  });
}
```

This pattern likely more useful than simply attempting to view the handle
during construction. Because handles at times need to have another operation
complete before the handle is fully constructed. For example if a hostname is
passed to `.listen()` then a DNS lookup needs to be performed. Until this
operation is complete no fd can be assigned to the `TCPWrap` instance.


### Constructor: `AsyncHook`

The `AsyncHook` constructor returns an instance that contains information about
the callbacks that are to fire during specific asynchronous events in the
lifetime of the event loop. The focal point of these calls centers around the
lifetime of `AsyncWrap`. These callbacks will also be called to emulate the
lifetime of handles and requests that do not fit this model. For example,
`HTTPPARSER` instances are recycled to improve performance. So the `destroy()`
callback would be called manually after a connection is done using it, just
before it's placed back into the unused resource pool.

All callbacks are optional. So if only resource cleanup needs to be tracked
then only the `destroy()` callback needs to be passed.

**Error Handling**: If any callback throws the application will print the stack
trace and exit. The exit path does follow that of any uncaught exception,
except for the fact that it is not catchable by an uncaught exception handler,
so any `'exit'` callbacks will fire. Unless the application is run with
`--abort-on-uncaught-exception`. In which case a stack trace will be printed
and the application will exit, leaving a core file.

The reason for this behavior is that these callbacks are running at potentially
volatile points in an object's lifetime. For example during class construction
and destruction. Because of this, it is deemed necessary to bring down the
process quickly as to prevent an unintentional abort in the future. This is
subject to change in the future if a comprehensive analysis is performed to
ensure an exception can follow the normal control flow without unintentional
side effects.


#### `asyncHook.scope()`

Enable capture of asynchronous events until the current synchronous code
execution has completed. This is basically a small amount of sugar to prevent
accidentally forgetting to `disable()` a set of hooks in a function with
multiple returns. Also in case an error is thrown and caught by a domain or
`uncaughtException`.

Say an application only wishes to observe the asynchronous calls made from
another asynchronous callback. Without `scope()` the call would look like so:

```js
fs.readFile(path, (err, data) => {
  asyncHook.enable();
  var e = asyncDataProcessing(data);
  if (e) {
    asyncHook.disable();
    return;
  }
  doMoreAsyncStuff();
  asyncHook.disable();
});
```

But using `scope()` all of that is handled for you:

```js
fs.readFile(path, (err, data) => {
  asyncHook.scope();
  var e = asyncDataProcessing(data);
  if (e)
    return;
  doMoreAsyncStuff();
});
```


#### `asyncHook.enable()`

Enable the callbacks for a given `AsyncHook` instance. Once a callback fires on
an asynchronous event they will continue to fire for all nested asynchronous
events. Even after the instance has been disabled.

Callbacks are not implicitly enabled after an instance is created. The reason
for this is to not make any assumptions about the user's use case. Since
constructing the `asyncHook` during startup, but not using it until later is, a
perfectly reasonable use case. This API is meant to err on the side of
requiring explicit instructions from the user.


#### `asyncHook.disable()`

Disable the callbacks for a given `AsyncHook` instance. Doing this will prevent
the `init()`, etc., calls from firing for any new roots of asynchronous call
stacks, but will not prevent existing asynchronous call stacks that have
already been captured by the `AsyncHook` instance from continuing to fire.

While not part of the immediate development plan, it should be possible in the
future to allow selective tracking of asynchronous call stacks. The following
example demonstrates this:

```js
const async_wrap = require('async_wrap');
const net = require('net');

// Pretend init, before, after are all defined
const asyncHook = async_wrap.createHook({ init, before, after });
asyncHook.enable();

net.createServer((c) => {
  // Only want to follow connections that match IP range.
  if (ipRangeRegExp.test(c.address().address))
    asyncHook.disable();
}).listen(PORT);
```

At which point no further calls to the hooks on that instance will be made for
that asynchronous branch.


### Hook Callbacks

Key events in the lifetime of asynchronous events have been categorized into
four areas. On instantiation, before/after the callback is called and when the
instance is destructed. For cases where resources are reused, instantiation and
destructor calls are emulated.


#### `init(id, type, parentId)`

* `id` {Number}
* `type` {String}
* `parentId` {Number}

Called when a class is constructed that has the possibility to trigger an
asynchronous event. This does not mean the instance will trigger a
`before()`/`after()` event before `destroy()` is called. Only that the
possibility exists.

This behavior can be observed by doing something like opening a resource then
closing it before the resource can be used. The following snippet demonstrates
this.

```js
require('net').createServer().listen(8080, function() { this.close() });
```

Every instance is assigned a unique id. Taking advantage of the fraction space
in the 64-bit IEEE 754 that ECMAScript defines as the **Number** type, the
number of id's that can be assigned are `2^53 - 1` (also defined as
`Number.MAX_SAFE_INTEGER`). At this size node can assign a new id every 100
nanoseconds and not run out for over 28 years. Because of this circumstance it
is not deemed necessary to find an alternative approach.

`parentId` is the unique id of either the async resource at the top of the JS
stack when `init()` was called, or the originator of the new resource. For
example, when a connection is made to a TCP server there is no JS stack
available when the connection's constructor is executed. Meaning there would be
no `parentId` available on the JS stack. Instead the TCP server's unique id is
manually passed to the client's constructor and propagated to the `init()`'s
`parentId`.


#### `before(id, entryPoint)`

* `id` {Number}
* `entryPoint` {Function}

Called just before the return callback is called after completing an
asynchronous request. Or called on handles with events such as receiving a new
connection. For requests, such as `fs.open()`, this should be called exactly
once. For handles, such as a TCP server, this may be called 0-N times.

The `entryPoint` is the callback that will be executed immediately after
`before()` returns. This is being passed as a way to track what work will be
done.

**Note(trevnorris):** I'm not sure passing only `entryPoint` is useful. Since
the arguments of the function to be called will be available it may be useful
to pass those to `before()` as well. Though I have reservations about the user
being able to mess with non-primitives.


#### `after(id, didThrow)`

* `id` {Number}
* `didThrow` {Boolean}

Called immediately after the return callback is completed. If the callback
threw but was caught by a domain or `uncaughtException`, `didThrow` will be set
to `true`. If the callback threw but was not caught then the process will exit
immediately without calling `after()`.


#### `destroy(id)`

* `id` {Number}

Called either when the class destructor is run, or if the resource is marked as
free. The destructor will usually run when explicitly called (the case for
handles) or when a request has completed. In several cases this will be
triggered by GC. In the case of shared or cached resources, such as
`HTTPParser`, `destroy()` will be called manually when the TCP connection is no
longer in need of it. Every subsequent use of a shared resource will have a new
unique id.


## API Exceptions

### net Client connection Event

Technically the `before()`/`after()` events of the `'connection'` event for
`net.Server` would place the server as the active id. Problem is that this is
not intuitive in how the asynchronous chain would propagate. So instead make
the client the active id for the duration of the `'connection'` callback.


### Reused Resources

Resources like `HTTPParser` are reused throughout the lifetime of the
application. This means node will have to synthesize the `init()` and
`destroy()` calls. Also the id on the class instance will need to be changed
every time the resource is acquired for use.

For shared resources like `TimerWrap` this is not necessary since there is a
unique JS handle that will contain the unique id necessary for the calls.


## Notes

### Promises

Node currently doesn't have sufficient API to notify calls to Promise
callbacks. In order to do so node would have to override the native
implementation.


### Immediate Write Without Request

When data is written through `StreamWrap` node first attempts to write as much
to the kernel as possible. If all the data can be flushed to the kernel then
the function exists without creating a `WriteWrap` and calls the user's callback
in `nextTick()`. Meaning the user would never be notified of data being written
because the entire operation was synchronous. Is this problematic?
