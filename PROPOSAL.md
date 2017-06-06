# Atomics.waitNonblocking (preliminary proposal)

Author: Lars T Hansen (lhansen@mozilla.com), April 2017

## Overview and background

We provide a new API, `Atomics.waitNonblocking`, that an agent can use
to wait on a shared memory location (to later be awoken by some agent
calling `Atomics.wake` on that location) without blocking.  Notably
this API is useful in agents whose `[[CanBlock]]` attribute is `false`,
such as the main thread of a web browser document, but the API is not
restricted to such agents.

The API is promise-based.  Very high performance is *not* a
requirement, but good performance is desirable.

This API was not included in ES2017 so as to simplify the initial API
of shared memory and atomics, but it is a desirable API for smooth
integration of shared memory into the idiom of ECMAScript as used in
web browsers.  A simple polyfill is possible but a native
implementation will likely have much better performance (both memory
footprint and execution time) than the polyfill.

Examples of usage: see `example.html` in this directory.

Prior history: Related APIs have been proposed before, indeed one
variant was in early drafts of the shared memory proposal under the
name `Atomics.futexWaitCallback`.


## Synopsis

`Atomics.waitNonblocking(i32a, index, value, [timeout]) => result`

The arguments are intepreted and checked as for `Atomics.wait`.  If
argument checking fails an exception is thrown synchronously, as for
`Atomics.wait`.

* `i32a` is an Int32Array mapping a SharedArrayBuffer
* `index` is a valid index within `i32a`
* `value` will be converted to Int32 and compared against the contents of `i32a[index]`
* `timeout`, if present, is a timeout value

The `result` is a promise.  The promise can be resolved with a string
value, one of `"ok"`, `"timed-out"`, `"not-equal"`; the value has the same
meaning as for the return type of `Atomics.wait`.  The promise is
never rejected.

(See "Performance and optimizations" below for a discussion of more
interesting result values.)

Agents can call `Atomics.wake` on some location corresponding to
`i32a[index]` to wake any agent waiting with
`Atomics.waitNonblocking`.  The agent performing the wake does not
need to know how that waiter is waiting, whether with `wait` or with
`waitNonblocking`.


## Informal semantics (aka notable facts)

Multiple agents can `waitNonblocking` on the same location at the same
time.  A `wake` on the location will resolve all the waiters'
promises, in some unspecified interleaving with unspecified
concurrency.

A single agent can `waitNonblocking` multiple times on a single
location before any of the waits are resolved.  A `wake` on the location
will resolve (in some arbitrary non-concurrent sequence) all the
promises.

Some agents can `wait` and other agents can `waitNonblocking` on the
same location at the same time, and a `wake` will wake all waiters
regardless of how they are waiting.

A single agent can first `waitNonblocking` and then, before that wait
is resolved, `wait` on a location.  A `wake` on the location will
first wake the agent from the second `wait`, and then resolve the
promise.

A single agent can `waitNonblocking` on a location and can then `wake`
on that location to resolve that promise within itself.

More generally, an agent can `waitNonblocking` on a location and only
subsequent to that take action that will cause some agent to perform a
`wake`.  For this reason, an implementation of `waitNonblocking` that
blocks is not viable (though see the Performance section).

Agents that wait with `waitNonblocking` participate in the same
fairness scheme as agents that wait with `wait`: When an agent
performs a `wake` with a count s.t. it does not wake all waiting
agents, waiting agents are woken in the order they waited.  (Note, the
preceding statement contradicts at least two of the preceding clauses
and is therefore too broad.  Resolving this conflict will be part of
pinning down the semantics for real.)

For practical purposes, we can think of the semantics as being those
of an implementation that creates a new helper agent for each
`waitNonblocking` call; this helper agent performs a normal blocking
`wait` on the location; and when the wait is complete, it sends an
asynchronous signal to the originating agent to resolve the promise
with the appropriate result value.

## Known open questions

* Whether `waitNonblocking` should always return a Promise or can
  return a Promise or a string.
* What we can and cannot require about wake order when wake order is
  observable.

## Polyfills

A simple polyfill is possible.  As suggested by the semantics, in the
Web domain it uses a helper Worker that performs a blocking `wait` on
behalf of the agent that is performing the `waitNonblocking`; that
agent and the helper communicate by message passing to coordinate
waiting and promise resolution.

As Workers are heavyweight and message passing is relatively slow, the
polyfill does not have excellent performance, but it is a reasonable
implementation and has good fidelity with the semantics.  (Helpers can
be reused within the agent that created them.)

The polyfill will not work in agents that cannot create new Worker
objects, either if they are too limited (worklets?) or if nested
Workers are not allowed (some browsers) or if a Worker cannot be
created from a data: URL.

See `polyfill.js` in this directory for the polyfill and
`example.html` for some test cases.


## Implementation challenges

It would seem that multiple implementation strategies are possible,
from having a thread per nonblocking wait (as the polyfill has) to
having no additional threads at all, instead dispatching runnables to
existing event loops in response to wakeups when records for
nonblocking waits are found in the wait/wake data structures (a likely
strategy for Firefox, for example).


## Performance and optimizations

For performance reasons it might appear that it is desirable to
"resolve the promise synchronously" if possible.  Leaving aside what
that would mean for a minute, here are some cases where the result of
the `waitNonblocking` could be available directly:

* The value in the array does not match the expected value and we can
  resolve synchronously with `"not-equal"`
* The value in the array matches and we have to sleep, but we want to
  micro-wait to see if a wakeup is received soon, in which case we can
  resolve synchronously with `"ok"`
* The value in the array matches but the timeout is zero, in which
  case we can resolve synchronously with `"timed-out"`

The first case is probably somewhat important.

The second case can backfire if the waiting agent needs to take action
to ensure the wakeup after creating the waiting promise.  But absent
that the case can be important for the performance of
producer-consumer problems involving a nonblocking thread.

The third case is not important; it is just a mystification of
`Atomics.load`.

Note that we can never resolve synchronously with `"timed-out"` if the
timeout is nonzero because we don't know if the waiting agent is going
to take action to perform the wakeup after setting up the wait.

Consider how this pattern would look with `await`.  First, in the case
where performance is not important, we don't even notice that there is
an optimization opportunity because `await` coerces the string to a
Promise:

```js
  switch(await Atomics.waitNonblocking(ia, idx, v)) {
    case "ok": ...
    case "timed-out": ...
    case "not-equal": ...
  }
```

And when we do care about the fast path, it's pretty sweet:

```js
  let r = Atomics.waitNonblocking(ia, idx, v);
  switch (r instanceof Promise ? (await r) : r) {
    case "ok": ...
    case "timed-out": ...
    case "not-equal": ...
  }
```

With plain promises it's messier, this is perhaps what the first case
would look like:

```js
  let r = Atomics.waitNonblocking(ia, idx, v);
  (r instanceof Promise ? r : Promise.resolve(r)).then(function (v) {
    switch (v) {
      case "ok": ...
      ...
    }
  })
```

And the second case might look like this:

```js
  let r = Atomics.waitNonblocking(ia, idx, v);
  let k = function (r) {
    switch (r) {
      case "ok" ...
      ...
    }
  }
  if (r instanceof Promise)
    r.then(k)
  else
    k(r)
```

Whether this change to the return type is a winner or not is partly a
matter of taste, partly a matter of usage patterns (which we don't
know yet), and partly a matter of how much performance we can hope to
wring out of the optimization in a credible implementation, ie, a
matter of how expensive it is to perform the non-blocking wait when a
wait is necessary, and how much there is to gain by avoiding it.  In
any case we must choose now.

For as much as I like this tweak I suspect that when performance
matters to that degree, the first case can be handled with an explicit
check preceding `waitNonblocking`, and in addition the implementation
can create a promise that resolves directly without involving any
actual waiting (effectively what `await` does).  For the second case
we could resuscitate the old idea of `Atomics.pause`, which allows for
controlled micro-waits in agents that otherwise cannot block; in
principle this provides better control.
