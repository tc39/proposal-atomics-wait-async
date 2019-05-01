# Atomics.waitAsync (Stage 2 proposal)

Author: Lars T Hansen (lhansen@mozilla.com), April 2017

## Overview and background

We provide a new API, `Atomics.waitAsync`, that an agent can use to
wait on a shared memory location (to later be awoken by some agent
calling `Atomics.wake` on that location) without waiting synchronously
(ie, without blocking).  Notably this API is useful in agents whose
`[[CanBlock]]` attribute is `false`, such as the main thread of a web
browser document, but the API is not restricted to such agents.

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

`Atomics.waitAsync(i32a, index, value, [timeout]) => result`

The arguments are interpreted and checked as for `Atomics.wait`.  If
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

Agents can call `Atomics.wake` on some location corresponding to
`i32a[index]` to wake any agent waiting with
`Atomics.waitAsync`.  The agent performing the wake does not
need to know how that waiter is waiting, whether with `wait` or with
`waitAsync`.


## Notable facts (informal semantics)

Multiple agents can `waitAsync` on the same location at the same time.
A `wake` on the location will resolve all the waiters' promises (as
many as the `count` argument to `wake` allows for).

A single agent can `waitAsync` multiple times on a single location
before any of the waits are resolved.  A `wake` on the location will
resolve (sequentially) all the promises (as many as the count argument
allows for).

Some agents can `wait` and other agents can `waitAsync` on the same
location at the same time, and a `wake` will wake waiters regardless
of how they are waiting.

A single agent can first `waitAsync` on a location and then, before
that wait is resolved, `wait` on the same location.

A single agent can `waitAsync` on a location and can then `wake` on
that location to resolve that promise within itself.

More generally, an agent can `waitAsync` on a location and only
subsequent to that take action that will cause some agent to perform a
`wake`.  For this reason, an implementation of `waitAsync` that blocks
is not viable.

There are two fairness schemes: inter-agent and intra-agent. Among different
agents, those that wait with `waitAsync` participate in the same fairness
scheme as those that wait with `wait`: when an agent performs a `notify` with a
`count` s.t. it does not wake all waiting agents, agents are notified in FIFO
order of waiting.

In the same agent, `waitAsync`s are notified in FIFO order, while any blocking
`wait`s are notified first. This is because an agent that is blocking in `wait`
cannot meaningfully resolve any promises returned by `waitAsync`.

## Semantics

See the [draft spec text](https://tc39.github.io/proposal-atomics-wait-async/).

## Polyfills

A simple polyfill is possible.

We can think of the semantics as being those of an implementation that
creates a new helper agent for each `waitAsync` call; this helper
agent performs a normal synchronous `wait` on the location; and when
the wait is complete, it sends an asynchronous signal to the
originating agent to resolve the promise with the appropriate result
value.

This polyfill models every situation, except for the
situation where an agent performs a `waitAsync` followed by a `wait`
and another agent subsequently asks to wake just one waiter - in
the real semantics, the `wait` is woken first, in the polyfill the
`waitAsync` is woken first.

As suggested by the semantics, in the Web domain it uses a helper
Worker that performs a synchronous `wait` on behalf of the agent that
is performing the `waitAsync`; that agent and the helper communicate
by message passing to coordinate waiting and promise resolution.

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
from having a thread per async wait (as the polyfill has) to
having no additional threads at all, instead dispatching runnables to
existing event loops in response to wakeups when records for
async waits are found in the wait/wake data structures (a likely
strategy for Firefox, for example).


## Performance and optimizations

### Synchronous resolution (rejected)

For performance reasons it might appear that it is desirable to
"resolve the promise synchronously" if possible.  Leaving aside what
that would mean for a minute, this was explored and the committee
decided it did not like the complexity of it.  Additionally, it's easy
enough to handle manually when it is necessary.  See SYNC-RESOLVE.md
for a writeup of the details around this idea.
