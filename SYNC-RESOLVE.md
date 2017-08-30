# Synchronous promise resolution

Note, this is a rejected performance idea, preserved for posterity.

There are some cases where the result of the `waitNonblocking` could
be available directly:

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

(Note that we can never resolve synchronously with `"timed-out"` if
the timeout is nonzero because we don't know if the waiting agent is
going to take action to perform the wakeup after setting up the wait.)

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
