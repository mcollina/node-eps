| Title  | AsyncHook API |
|--------|---------------|
| Author | @trevnorris   |
| Status | DRAFT         |
| Date   | 2016-09-07    |

## Description

Since its initial introduction along side the `AsyncListener` API, the internal
class `AsyncWrap` has slowly evolved to ensure a generalized API that would
serve as a solid base for module authors who wished to add listeners to the
event loop's life cycle. Some of the use cases `AsyncWrap` has covered are long
stack traces, continuation local storage, profiling of asynchronous requests
and resource tracking. Though the public API is now exposed as `'async_hooks'`.

It must be clarified that the `AsyncHook` API is not meant to abstract away
Node's implementation details, and is in fact intentional. Observing how
operations are performed, not simply how they appear to perform from the public
API, is a key point in its usability. For a real world example, a user was
having difficulty figuring out why their `http.get()` calls were taking so
long. Because `AsyncHook` exposed the `dns.lookup()` request to resolve the
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

Performance impact of `AsyncHook` should be zero if not being used, and near
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
const async_hooks = require('async_hooks');

// Return the id of the current execution context. Useful for tracking state
// and retrieving the handle of the current parent without needing to use an
// AsyncHook.
const id = async_hooks.currentId();

// Create a new instance of AsyncHook. All of these callbacks are optional.
const asyncHook = async_hooks.createHook({ init, before, after, destroy });

// Allow callbacks of this AsyncHook instance to fire. This is not an implicit
// action after running the constructor, and must be explicitly run to begin
// executing callbacks.
asyncHook.enable();

// Disable listening for new asynchronous events. Though this will not prevent
// callbacks from firing on asynchronous chains that have already run within
// the scope of an enabled AsyncHook instance.
asyncHook.disable();

// Unlike disable(), prevent any hook callback from firing again in the future.
// Future hooks can be added by running syncScope()/enable() again.
// NOTE: Unsure this will make it into the initial release.
asyncHook.remove();

// The following are the callbacks that can be passed to createHook().

// init() is called during object construction. The handle may not have
// completed construction when this callback runs. So all fields of the
// handle referenced by "id" may not have been populated.
function init(id, type, parentId, handle) { }

// before() is called just before the handle's callback is called. It can be
// called 0-N times for handles (e.g. TCPWrap), and should be called exactly 1
// time for requests (e.g. FSReqWrap).
function before(id) { }

// after() is called just after the handle's callback has finished, and will
// always fire. Except in the case of an uncaught exception.
function after(id) { }

// destroy() is called when an AsyncWrap instance is destroyed. In cases like
// HTTPParser where the resource is reused, or timers where the handle is only
// a JS object, destroy() will be triggered manually soon after after() has
// completed.
function destroy(id) { }

// The following calls are specific to the embedder API. If any hook emitter is
// used then they must all be used to make sure state proceeds correctly.

// Return new unique id for a constructing handle.
const id = async_hooks.newUid();

// Call the init() callbacks. Returns an instance of AsyncEvent that will be
// used in other emit calls.
const event = async_hooks.emitInit(id, handle, type[, parentId[, parentEvent]]);

// Call the before() callbacks. The reason for requiring both arguments is
// explained in further detail below.
async_hooks.emitBefore(id, event);

// Call the after() callbacks.
async_hooks.emitAfter(id, event);

// Call the destroy() callbacks.
async_hooks.emitDestroy(id, event);
```


### `async_hooks`

The object returned from `require('async_hooks')`.


#### `async_hooks.currentId()`

Return the id of the current execution context. Useful to track the
asynchronous execution chain. It can also be used to propagate state without
needing to use the `AsyncHook` constructor.


#### `async_hooks.createHook(callbacks)`

* `callbacks` {Object}
* Return: {AsyncHook}

`createHook()` returns an `AsyncHook` instance that contains information about
the callbacks that are to fire during specific asynchronous events in the
lifetime of the event loop. The focal point of these calls centers around the
lifetime of the `AsyncWrap` C++ class. These callbacks will also be called to
emulate the lifetime of handles and requests that do not fit this model. For
example, `HTTPParser` instances are recycled to improve performance. So the
`destroy()` callback would be called manually after a connection is done using
it. Just before it's placed back into the unused resource pool.

All callbacks are optional. So, for example, if only resource cleanup needs to
be tracked then only the `destroy()` callback needs to be passed. The
specifics of all functions that can be passed to `callbacks` is in the section
`Hook Callbacks`. (TODO(trevnorris): make this a link)

**Error Handling**: If any `AsyncHook` callbacks throw the application will
print the stack trace and exit. The exit path does follow that of any uncaught
exception, except for the fact that it is not catchable by an uncaught
exception handler, so any `'exit'` callbacks will fire. Unless the application
is run with `--abort-on-uncaught-exception`. In which case a stack trace will
be printed and the application will exit, leaving a core file.

The reason for this error handling behavior is that these callbacks are running
at potentially volatile points in an object's lifetime. For example during
class construction and destruction. Because of this, it is deemed necessary to
bring down the process quickly as to prevent an unintentional abort in the
future. This is subject to change in the future if a comprehensive analysis is
performed to ensure an exception can follow the normal control flow without
unintentional side effects.


#### `asyncHook.enable()`

* Return: {AsyncHook} A reference to `asyncHook`.

Enable the callbacks for a given `AsyncHook` instance. Once a callback fires on
an asynchronous event they will continue to fire for all nested asynchronous
events. An `asyncHook` is automatically disabled at the end of any synchronous
call stack. The reason for this is because when the following synchronous call
stack begins it will have its own state to load. Even if that state contains
nothing.

Though all propagated hooks will continue to fire along the asynchronous call
stack. Even if `asyncHook` was manually disabled.

```js
const async_hooks = require('async_hooks');
const hooks = new async_hooks.AsyncHook({ init });
hooks.enable();

net.createServer((c) => {
  // Even though the hooks have been disabled by the time this callback runs
  // the callbacks attached to "hooks" will still fire.
}).listen(8080);

hooks.disable();
```

Callbacks are not implicitly enabled after an instance is created. The reason
for this is to not make any assumptions about the user's use case. Since
constructing the `asyncHook` during startup, but not using it until later is, a
reasonable use case. This API is meant to err on the side of requiring explicit
instructions from the user.

To help simplify this `enable()` returns the instance of itself so the calls
can be chained.

```js
const async_hooks = require('async_hooks');

async_hooks.createHook(callbacks).enable();
```


#### `asyncHook.disable()`

Disable the callbacks for a given `AsyncHook` instance. Doing this will prevent
the `init()`, etc., calls from firing for any new roots of asynchronous call
stacks, but will not prevent existing asynchronous call stacks that have
already been captured by the `AsyncHook` instance from continuing to fire.

Be careful to always disable the hooks when they are no longer needed.
Especially in cases where there are multiple return statements. If the hooks
are not disabled then they will stay active indefinitely. The following is an
example of how a call to `disable()` may accidentally left out:

```js
const hooks = async_hooks.createHook(/* ... */);

crypto.randomBytes(1024, (err, data) => {
  hooks.enable();
  try {
    fs.writeFileSync(path, data);
  } catch (e) {
    console.error('file write failed');
    // Notice the early return and failure to run hooks.disable(), which will
    // allow the hooks callbacks to continue firing indefinitely.
    return;
  }
  hooks.disable();
});
```

It is possible to have selective tracking of asynchronous call stacks. The
following example demonstrates this:

```js
const async_hooks = require('async_hooks');
const net = require('net');

// Pretend init, before, after are all defined
const asyncHook = async_hooks.createHook({ init, before, after });
asyncHook.enable();

net.createServer((c) => {
  // Only want to follow connections that match IP range.
  if (!ipRangeRegExp.test(c.address().address))
    asyncHook.disable();
}).listen(PORT);
```

At which point no further calls to the hooks on that instance will be made for
that asynchronous branch.


#### `asyncHook.remove()`

NOTE: While planned to be implemented, will probably not make the initial
release.

Unlike `disable()`, `remove()` prevents the callbacks from firing again.
Regardless of whether they have been attached to an asynchronous execution
chain or not.

```js
net.createServer((c) => {
  asyncHook.syncScope();
  c.on('data', (chunk) => {});
  c.on('end', () => {});
});

setTimeout(() => {
  // All new connections that have attached asyncHooks created in the last five
  // seconds will not fire again.
  asyncHook.remove();
}, 5000);
```

After `remove()` is run the `asyncHook` will operate normally, and as if it had
never run `enable()` in the past


### Hook Callbacks

Key events in the lifetime of asynchronous events have been categorized into
four areas. On instantiation, before/after the callback is called and when the
instance is destructed. For cases where resources are reused, instantiation and
destructor calls are emulated.


#### `init(id, type, parentId, handle)`

* `id` {Number}
* `type` {String}
* `parentId` {Number}
* `handle` {Object}

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
is not deemed necessary to use an alternative approach that would account for
the id to wrap around.

The `type` is a String that represents the type of handle that caused `init()`
to fire. Generally it will be the name of the handle's constructor. Some
examples include `TCP`, `GetAddrInfo` and `HTTPParser`. Users will be able to
define their own `type` when using the public embedder API.

`parentId` is the unique id of either the async resource at the top of the JS
stack when `init()` was called, or the originator of the new resource. For
example, when a connection is made to a TCP server there is no JS stack
available when the connection's constructor is executed. Meaning there would be
no `parentId` available on the JS stack. Instead the TCP server's unique id is
manually passed to the client's constructor and propagated to the `init()`'s
`parentId`.

The constructed `handle` is passed to `init()`. The structure of any object
passed to `init()` has no guarantee of stability, even through patch updates.
It's meant to be used for debugging the application when additional, and
specific, information is needed. Make sure not to hold a reference to this
indefinitely or else the GC won't be able to collect it after node has removed
its own reference to the handle.


#### `before(id)`

* `id` {Number}

Called just before the return callback is called after completing an
asynchronous request. Or called on handles with events such as receiving a new
connection. For requests, such as `fs.open()`, this should be called exactly
once. For handles, such as a TCP server, this may be called 0-N times.


#### `after(id)`

* `id` {Number}

Called immediately after the return callback is completed. If a callback throws
but is not caught then the process will exit immediately without calling
`after()`.


#### `destroy(id)`

* `id` {Number}

Called either when the class destructor is run, or if the resource is marked as
free. The destructor will usually run when explicitly called (the case for
handles) or when a request has completed. In several cases this will be
triggered by GC. In the case of shared or cached resources, such as
`HTTPParser`, `destroy()` will be called manually when the TCP connection is no
longer in need of it. Every subsequent use of a shared resource will have a new
unique id.


## Embedder API

Library developers that handle their own I/O will need to hook into the
`AsyncWrap` API so that all the appropriate callbacks are called. To
accommodate this both a C++ and JS API is provided.


### Native API

```cpp
// Helper class users can inherit from, but is not necessary. It will
// automatically call all four callbacks.
class AsyncHook {
  public:
    AsyncHook(Local<Object> handle, const char* name);
    ~AsyncHook();
    MaybeLocal<Value> MakeCallback(
        const Local<Function> callback,
        int argc,
        Local<Value>* argv);
    Local<Object> handle();
    uint64_t get_uid();
  private:
    AsyncHook();
    Persistent<Object> handle_;
    const char* name_;
    uint64_t uid_;
}

// Returns the id of the current execution context. If the return value is
// zero then no execution has been set. This will happen if the user handles
// I/O from native code.
uint64_t node::GetCurrentId();

// If the native API doesn't inherit from the helper class then the callbacks
// must be triggered manually. This triggers the init() callback. The return
// value is the uid assigned to the handle.
// TODO(trevnorris): This always needs to be called so that the handle can be
// placed on the Map for future query.
uint64_t node::EmitAsyncHandleInit(Local<Object> handle);

// Emit the destroy() callback.
void node::EmitAsyncHandleDestroy(uint64_t id);

// An API specific to emit before/after callbacks is unnecessary because
// MakeCallback will automatically call them for you.
MaybeLocal<Value> node::MakeCallback(Isolate* isolate,
                                     Local<Object> recv,
                                     Local<Function> callback,
                                     uint64_t id,
                                     int argc,
                                     Local<Value>* argv);
```


### JS API

There is also a JS API to allow for conceptual correctness. For example, if a
request batches calls under the hood this would still allow for each request
to trigger the callbacks individually so that the callback associated with
each bach request execute inside the correct parent.

This cannot be enforced in any way. It is up to the implementer to make sure
all of the callbacks are placed and called at the correct time.


#### `async_hooks.newUid()`

* Return: {Number}

Return a new unique id for a given handle from the same pool of unique id's
used for all internally created handles.

Generally this should be used during object construction. e.g.:

```js
class MyClass {
  constructor() {
    this._event_id = async_hooks.newUid();
  }
}
```


#### `async_hooks.emitInit(id, handle, type[, parentId[, parentEvent]])`

* `id` {Number} Generated by calling `newUid()`
* `handle` {Object}
* `type` {String}
* `parentId` {Number} **Default:** `currentId()`
* `parentEvent` {Object} **Default:** `undefined`
* Return: {Object|Null}

Emit that a handle is being initialized. `id` should be a value returned by
`async_hooks.newUid()`. Usage will probably be as follows:

```js
class Foo {
  constructor() {
    this.event_id = async_hooks.newUid();
    this.event_stor = async_hooks.emitInit(this.event_id, this, 'Foo');
  }
}
```

The return value from `emitInit()` is either an instance of the internal
constructor `AsyncEventStor`. Which will contain the state of all hooks that
must propagate to future async calls. If there are no hooks that need to
propagate then the return value will be `null`. Preventing the need of creating
one additional object for every async constructor, even when hooks aren't being
used.

In the unusual circumstance that the embedder needs to define a different
parent id than `currentId()`, they can pass in that id manually. Along with
the `AsyncEventStor`, if there is one. If no stor is passed then it is assumed
there are no hooks that need to propagate to the newly created stor.

It is suggested to have `emitInit()` be the last call in the object's
constructor.


#### `async_hooks.emitBefore(id, event)`

* `id` {Number} Generated by `newUid()`
* `event` {AsyncEventStor}
* Return: {Undefined}

Trigger listening `before()` callbacks that the `handle` is about to enter it's
call stack.


#### `async_hooks.emitAfter(id, event)`

* `id` {Number} Generated by `newUid()`
* `event` {AsyncEventStor}
* Return: {Undefined}

Trigger listening `after()` callbacks that the `handle` has left it's call
stack.

The `id` call stack should always properly unwind. No two `emitAfter()` calls
should be crossed. For example:

```
init # Foo
init # Bar
...
before # Foo
before # Bar
after # Foo   <- Should be called after Bar
after # Bar
```

There is no known scenario where this is acceptable. If one is found then this
restriction may be lifted.


#### `async_hooks.emitDestroy(id, event)`

* `id` {Number} Generated by `newUid()`
* `event` {AsyncEventStor}
* Return: {Undefined}

Trigger listening `destroy()` callbacks that the `handle`'s resources are no
longer being used, and in cases where the `handle` is attached to native
resources the underlying class may have been destructed.


## API Exceptions

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
the function exists without creating a `WriteWrap` and calls the user's
callback in `nextTick()`. Meaning detection of the write won't be as
straightforward as watching for a `WriteWrap`.
