# Atomics.waitAsync (Stage 1 proposal)

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
many as the count argument allows for).

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

Agents that wait with `waitAsync` participate in the same fairness
scheme as agents that wait with `wait`: When an agent performs a
`wake` with a count s.t. it does not wake all waiting agents, waiting
agents are woken in a predictable order.


## Semantics

Several changes to the existing spec are necessary to accomodate
`Atomics.asyncWait`.  These changes are not in themselves intended to
change semantics.

Primarily, timeouts are made less magical by introducing a notion of
concurrent "alarms" which are thunks that are run in response to
expired timeouts and perform wakeups using the standard mechanisms.
Alarms are named by IDs and sets of these IDs hang off the
WaiterLists.  The sets are in some sense optional, but clarify the
coordination between alarms and explict wakeups as well as the
lifetime of the alarm thunks.

Secondarily, the job of removing a waiter has been moved to the thread
performing a wakeup, whether that is another agent performing a `wake`
or the concurrent alarm thread.

With that background, the changes necessary to add `asyncWait` boil
down to generalizing the WaiterList to hold both sync and async
waiters, and to inserting waiters into that list by priority.


24.4.1.3, GetWaiterList

Change this section as follows.

Before the first paragraph, insert this paragraph:

"A _Waiter_ is a pair (_agent_, false | _promise_) where _agent_ is
the agent signifier of the agent that is waiting.  Agents that are
waiting in `Atomics.wait` will be designated by a waiter where the
second element of the pair is false; agents that are waiting in
`Atomics.asyncWait` will be designated by a waiter where the second
element of the pair is the promise that is to be resolved when the
agent is awoken."

In the current first paragraph, change the word "agent" to "waiter".

After the current first paragraph, insert these two paragraphs:

"There can be multiple entries in a _WaiterList_ with the same agent
signifier, but at most one of those will have false in the second
element of the entry."

"The _WaiterList_ has an attached _alarm set_, a set of truthy values.
This set is manipulated only when the thread manipulating it is in the
critical section for the _WaiterList_."

In the current third paragraph, add "adding and removing alarms" to
the list of guarded operations.

Spec note: The alarm may perhaps be further formalized so that the
clause about the possibly absent alarm ID is not needed in the spec of
CancelAlarm, but I'm not sure this would actually clarify anything.


24.4.1.6, AddWaiter( _WL_, _W_, _promise_ )

The abstract operation AddWaiter takes three arguments, a WaiterList
_WL_, an agent signifier _W_, and _promise_, which is either false or
a Promise object. It performs the following steps:

1. Assert: The calling agent is in the critical section for _WL_.
1. Assert: (_W_, _promise_) is not on the list of waiters in any WaiterList.
1. If _promise_ is false:
  1. Insert (_W_, false) into the list of waiters in _WL_ before any other entry (_W_, _x_) in the list.
1. Else,
  1. Add (_W_, _promise_) to the end of the list of waiters in _WL_.


24.4.1.7, RemoveWaiter( _WL_, _W_, _promise_ )

The abstract operation RemoveWaiter takes three arguments, a
WaiterList _WL_, an agent signifier _W_, and _promise_, which is
either false or a Promise object. It performs the following steps:

1. Assert: The calling agent is in the critical section for _WL_.
1. Assert: (_W_, _promise_) is on the list of waiters in _WL_.
1. Remove (_W_, _promise_) from the list of waiters in _WL_. 


24.4.1.9, Suspend( _WL_, _W_)

Change this function not to take a 'timeout' argument.  Timeouts are
now handled in the caller.  (Not intended as a semantic change.)

Change step 5 to be this:

1. Perform LeaveCriticalSection(WL) and suspend W, performing the combined operation in such a way that a wakeup that arrives after the critical section is exited but before the suspension takes effect is not lost. W can wake up only because it was woken explicitly by another agent calling WakeWaiter(WL, W), not for any other reasons at all.

Remove steps 7 and 8.


24.4.1.10, WakeWaiter( _WL_, _W_, _promise_ )

The abstract operation WakeWaiter takes three arguments, a WaiterList
_WL_, an agent signifier _W_, and _promise_, which is either false or
a Promise object. It performs the following steps:

1. Assert: The calling agent is in the critical section for _WL_.
1. Assert: (_W_, _promise_) is on the list of waiters in _WL_.
1. If _promise_ is false:
  1. Wake the agent _W_. 
1. Else,
  1. Make _promise_ resolveable.  (Spec details TBD.)


24.4.1.13, AddAlarm(_WL_, _alarmFn_, _timeout_)

Spec note: A new section.

AddAlarm takes three arguments, a WaiterList _WL_, a thunk _alarmFn_,
and a nonnegative finite number _timeout_.  It performs the following
steps:

1. Assert: the current thread is in the critical section on _WL_.
1. Let _alarm_ be a truthy value that is not in _WL_'s alarm set.
1. Add _alarm_ to _WL_'s alarm set.
1. After _timeout_ milliseconds has passed, perform the following actions on a concurrent thread:
  1. Perform EnterCriticalSection(_WL_).
  1. If _alarm_ is in _WL_'s alarm set:
    1. Remove _alarm_ from _WL_'s alarm set.
    1. Perform _alarmFn_().
  1. Perform LeaveCriticalSection(_WL_).
  1. Note: _alarmFn_ is now dead.
1. Return _alarm_.


24.4.1.4, CancelAlarm(_WL_, _alarm_)

Spec note: A new section.

CancelAlarm takes two arguments, a WaiterList _WL_ and an alarm ID
_alarm_.  It performs the following steps:

1. Assert: the current thread is in the critical section on _WL_
1. Assert: _alarm_ is not false.
1. Remove _alarm_ from _WL_'s alarm set (it may not be present).
1. Note: the thunk associated with _alarm_ is now dead.


24.4.11, Atomics.wait(_typedArray_, _index_, _value_, _timeout_)

Replace Steps 16-19 with the following:

1. Perform AddWaiter(_WL_, _W_, false).
1. Let _awoken_ be true.
1. Let _alarm_ be false.
1. If _q_ is finite then:
  1. Let _alarmFn_ be a function of no arguments that does the following:
    1. Set _awoken_ to false.
    1. Perform RemoveWaiter(_WL_, _W_, false)
    1. Perform WakeWaiter(_WL_, _W_, false).
  1. Set _alarm_ to AddAlarm(_WL_, _alarmFn_, _q_).
1. Perform Suspend(_WL_, _W_)
1. If _awoken_ is true and _alarm_ is not false:
  1. Perform CancelAlarm(_WL_, _alarm_)
1. Assert: (_W_, false) is not on the list of waiters in WL.


24.4.12, Atomics.wake(_typedArray_, _index_, _count_)

Modify this algorithm as follows:

In step 12.a., change the word _agent_ to the work _waiter_.

(RemoveWaiters no longer returns a list of pairs, but a list of
waiters, which are (_agent_, false | Promise) pairs.)


24.4.15, Atomics.asyncWait( _typedArray_, _index_, _value_, _timeout_ )

Spec note: A new section.  This is substantially similar to Atomics.wait.

1. Let _buffer_ be ? ValidateSharedIntegerTypedArray(_typedArray_, true).
1. Let _i_ be ? ValidateAtomicAccess(_typedArray_, _index_).
1. Let _v_ be ? ToInt32(_value_).
1. Let _q_ be ? ToNumber(_timeout_).
1. If _q_ is NaN, let _t_ be + infinity, else let _t_ be max(_q_, 0).
1. Let _block_ be _buffer_.[[ArrayBufferData]].
1. Let _offset_ be _typedArray_.[[ByteOffset]].
1. Let _indexedPosition_ be (_i_ × 4) + _offset_.
1. Let _WL_ be GetWaiterList(_block_, _indexedPosition_).
1. Perform EnterCriticalSection(_WL_).
1. Let _w_ be ! AtomicLoad(_typedArray_, _i_).
1. If _v_ is not equal to _w_, then
  1. Perform LeaveCriticalSection(_WL_).
  1. Return a new Promise resolved with the String "not-equal" (as for Promise.resolve)
1. Let _W_ be AgentSignifier().
1. Let _awoken_ be true.
1. Let _alarm_ be false.
1. Let _executor_ be a function of two arguments, _resolve_ and _reject_, that does the following:
  1. If _awoken_ is true then
    1. If _alarm_ is not false:
      1. Perform CancelAlarm(_WL_, _alarm_)
    1. Invoke _resolve_ on the string "ok"
  1. else
    1. Invoke _resolve_ on the string "timed-out"
1. Let _P_ be a new Promise created with the executor _executor_
1. If _q_ is finite:
  1. Let _alarmFn_ be a function of no arguments that does the following:
    1. Set _awoken_ to true.
    1. Perform RemoveWaiter(_WL_, _W_, _P_)
    1. Perform WakeWaiter(_WL_, _W_, _P_)
  1. Set _alarm_ to AddAlarm(_WL_, _alarmFn_, _q_)
1. Perform AddWaiter(_WL_, _W_, _P_)
1. Perform LeaveCriticalSection(_WL_).
1. Return _P_.


## Polyfills

A simple polyfill is possible.

We can think of the semantics as being those of an implementation that
creates a new helper agent for each `waitAsync` call; this helper
agent performs a normal synchronous `wait` on the location; and when
the wait is complete, it sends an asynchronous signal to the
originating agent to resolve the promise with the appropriate result
value.

This polyfill models every situation, POSSIBLY except for the
situation where an agent performs a `waitAsync` followed by a `wait`
and another agent subsequently asks to wake just one waiter - MAYBE in
the real semantics, the `wait` is woken first, in the polyfill the
`asyncWait` is woken first.  NOTE: exact real semantics TBD.

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
