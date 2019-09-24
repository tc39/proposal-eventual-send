# Eventual-Send: Support for distributed promise pipelining
By Mark S. Miller (@erights), Chip Morningstar (@FUDCo), and Michael FIG (@michaelfig)

**ECMAScript Eventual-Send: Support for distributed promise pipelining**

## Background

Promises were invented in the late 1980s, originally as a technique for
compensating for roundtrip latency in operations invoked remotely over a
network, though promises have since proven valuable for dealing with all manner
of asynchronous delays in computational systems.

The fundamental insight behind promises is this: in the classic presentation of
object oriented programming, an object is something that you can send messages
to in order to invoke operations on it.  If the result of such an operation is
another object, that result in turn is something that you can send messages to.
If the operation initiated by a message entails an asynchronous delay to get
the result, rather than forcing the sender to wait (possibly a long time) for
the result to eventually become available, the system can instead immediately
return another object - a promise - that can stand in for the result in the
meantime.  Since, as was just said, an object is something you send messages
to, a promise is, in that respect, potentially as good as the object it is a
promise for -- you simply send it messages as if it was the actual result.  The
promise can't perform the invoked operation directly, since what that means is
not yet known, but it *can* enqueue the request for later processing or relay
it to the other end of a network connection where the result will eventually be
known.  This deferall of operations through enqueuing or relaying can be
pipelined an arbitrary number of operations deep; it is only at the point where
there is a semantic requirement to actually see the result (such as the need to
display it to a human) that the pipeline must stall to await the final outcome.
Furthermore, experience with this paradigm has shown that the point at which
such waiting is truly required can often be much later in a chain of
computational activity than many people's intuitions lead them to expect.

Since network latency is often the largest component of delay in a remotely
invoked operation, the overlapping of network transmissions that promise
pipelining makes possible can result an enormous overall improvement in
throughput in distributed systems.  For example, implementations of promise
pipelining for remote method invocation in the [Xanadu hypertext
system](http://udanax.xanadu.com/gold/) and in Microsoft's [Midori operating
system](http://joeduffyblog.com/2015/11/03/blogging-about-midori/) measured
speedups of 10 to 1,000 over traditional synchronous RPC, depending on use
case.

Promises in JavaScript were proposed in the 2011 [ECMAScript strawman
concurrency
proposal](https://web.archive.org/web/20161026162206/http://wiki.ecmascript.org/doku.php?id=strawman:concurrency).
These promises descend from the [E language](http://erights.org/) via the
[Waterken Q library](http://waterken.sourceforge.net/web_send/) and [Kris
Kowal's Q library](https://github.com/kriskowal/q).  A good early presentation
is Tom Van Cutsem's [Communicating Event Loops: An exploration in
JavaScript](http://soft.vub.ac.be/~tvcutsem/talks/presentations/WGLD_CommEventLoops.pdf).
All of these efforts introduced promises as a first step towards distributed
computing, with the goal of using promises as asynchronous references to remote
objects.  However, since the JavaScript language itself does not contain any
intrinsic I/O machinery, relying entirely on the host environment for this,
Promises as JavaScript currently defines them are not by themselves sufficient
to realize the distributed computation vision that originally motivated them.

Kris Kowal's [Q-connection library](https://github.com/kriskowal/q-connection)
extended Q's promises for distributed computing with [promise
pipelining](https://capnproto.org/rpc.html), essentially in the way we have in
mind.  However, in the absence of platform support for [Weak
References](https://github.com/tc39/proposal-weakrefs), this approach was not
practical.  Given weak references, the [Midori
project](http://joeduffyblog.com/2015/11/19/asynchronous-everything/) and
[Cap'n Proto](https://capnproto.org/rpc.html), among others, demonstrate that
this approach to distributed computing works well at scale.

## Summary

This proposal adds *eventual-send* operations to JavaScript Promises, to
express invocation of operations on potentially remote objects.  We introduce
the notion of a *handled Promise*, whose handler can provide alternate
eventual-send behavior.  These mechanisms, together with weak references,
enable the creation of remote object communications systems, but without
committing to any specific implementation.  In particular, this proposal
specifies a general mechanism for hooking in whatever host-provided remote
communications facilities are at hand, without constraining the nature of those
facilities.

This proposal does not mandate any specific usage of the mechanisms it
describes.  Such usages as are mentioned here are provided as explanatory and
motivating examples and as ways testing the adequacy of the design, rather than
proposing a particular implementation of remote messaging.


## Design Principles

1. Support *promise pipelining* to reduce the cost of network latency.
1. Prevent reentrancy attacks (a form of plan interference).

## Details

To specify eventual-send operations and handled promises, we follow the pattern
used to incorporate proxies into JavaScript:  That pattern specified...

   * internal methods that all objects must support.
   * static `Reflect` methods for invoking these internal methods.
   * invariants that these methods must uphold.
   * default behaviors of these methods for normal (non-exotic) objects.
   * how proxies implement these methods by delegating most of their behaviors to corresponding traps on their handlers.
   * the remaining behavior in the proxy methods to guarantee that these invariants are upheld despite arbitrary behavior by the handler.
   * fallback behaviors for absent traps, implemented in terms of the remaining traps.

Following this analogy, this proposal adds internal eventual-send methods
to all promises, provides default behaviors for unhandled promises, and
introduces handled promises whose handlers provide traps for these methods.

A new constructor, `HandledPromise`, enables the creation of handled
promises. The static methods below are static methods of this constructor.

| Internal Method | Static Method |
| --- | --- |
| `p.[[GetSend]](prop)` | `get(p, prop)` |
| `p.[[SetSend]](prop, value)` | `set(p, prop, value)` |
| `p.[[DeleteSend]](prop)` | `delete(p, prop)` |
| `p.[[ApplySend]](args)` | `apply(p, args)` |
| `p.[[ApplyMethodSend]](prop, args)`| `applyMethod(p, prop, args)` |

The static methods first do the equivalent of `Promise.resolve` on their first
argument, to coerce it to a promise with these internal methods.  Thus, for
example,

```js
HandledPromise.get(p, prop)
```
actual does the equivalent of
```
Promise.resolve(p).[[GetSend]](prop)
```

Via the internal methods, the static methods cause either the default behavior,
or, for handled promises, the behavior that calls the associated handler trap.

| Static Method | Default Behavior | Handler trap |
| --- | --- | --- |
| `get(p, prop)` | `p.then(t => t[prop])` | `h.get(t, prop)` |
| `set(p, prop, value)` | `p.then(t => (t[prop] = value))` | `h.set(t, prop, value)` |
| `delete(p, prop)` | `p.then(t => delete t[prop])` | `h.delete(t, prop)` |
| `apply(p, args)` | `p.then(t => t(...args))` | `h.apply(t, args)` |
| `applyMethod(p, prop, args)` | `p.then(t => t[prop](...args))` | `h.applyMethod(t, prop, args)` |

To protect against reentrancy, the proxy internal method postpones the
execution of the handler trap to a later turn, and immediately returns a
promise for what the trap will return.  For example, the [[GetSend]] internal
method of a handled promise is effectively

```js
p.then(t => h.get(t, prop))
```

Sometimes, these operations will be used to cause remote effects while ignoring
the local promise for the result.  For distributed messaging protocols, the
extra bookkeeping for these return results are sufficiently expensive that we
should be able to avoid it when it is not needed.  To support this, we
introduce the "SendOnly" variants of these methods.  For example, the SendOnly
variant of the [[Get]] trap looks like this:

| Internal Method | Static Method | Default Behavior | Handler trap |
| --- | --- | --- | --- |
| `p.[[GetSendOnly]](prop)` | `getSendOnly(p, prop)` | `void p.then(t => t[prop])` | `h.getSendOnly(t, prop)` |

The others ([[SetSendOnly]], [[ApplySendOnly]], etc.) all follow exactly the
same pattern.  We will collectively refer to these as the "\*SendOnly"
operations.

No matter what a \*SendOnly handler trap returns, the proxy internal
[[\*SendOnly]] method always immediately returns `undefined`.

When a "SendOnly" trap is absent, the trap behavior defaults to the
corresponding non-SendOnly trap.  But again, the proxy internal [[\*SendOnly]]
method always immediately returns `undefined`, and so is effectively, for
example:

```js
void p.then(t => h.get(t, prop))
```

### `E` and `E.sendOnly` convenience proxies

Probably the most common distributed programming case, invocation of remote
methods with or without requiring return results, can be implemented by
powerless proxies.  All authority needed to enable communication between the
peers can be implemented in the handled promise infrastructure.

The `E(target)` proxy maker wraps a remote target and allows for a single
remote method call returning a promise for the result.

```js
E(target).method(arg1, arg2...) // Promise<result>
```

`E.sendOnly(target)` is similar, but declares that we do not want the result
(or even acknowledgement).  It always immediately returns `undefined`.

Example usage:

```js
import { E } from 'js:eventual-send';

// Invoke pipelined RPCs.
const fileP = E(
  E(target).openDirectory(dirName)
).openFile(fileName);
// Process the read results after a round trip.
E(fileP).read().then(contents => {
  console.log('file contents', contents);
  // We don't use the result of this send.
  E.sendOnly(fileP).append('fire-and-forget');
});
```

*{{This portion of the proposal is unclear.  (It's not event entirely obvious
 that it's part of the proposal at all.)  Perhaps this would be clarified by
 providing an example implementation for `E`?}}*

### `HandledPromise` constructor

In a manner analogous to *Proxy* handlers, a **handled promise** is associated
with a handler object.

For example,

```js
import { HandledPromise } from 'js:eventual-send';

// create a handled promise with initial handler:
new HandledPromise((resolve, reject) => ..., unfulfilledHandler);
// designate a different handler after resolution:
resolve(presence, fulfilledHandler)
// or use the same handler as myPromise
resolve(myPromise)

// the default unfulfilledHandler queues messages until resolution.
new HandledPromise((resolve, reject) => ...)
```

*{{This example is confusing in that it is not a single coherent block of code
 but rather a bunch of individual lines that look as if they go together but
 don't.  This is particularly confusing following the prior code block example
 which* was *a single coherent body.}}*

This handler is not exposed to the user of the handled promise, so it provides
a secure separation between the unprivileged client (which uses the `E`,
`E.sendOnly` or static `HandledPromise` methods) and the privileged system
which implements the communication mechanism.

### `HandledPromise.prototype`

Although `HandledPromise` is class-like, it is not intended to act like a class
distinct from `Promise`.  The initial value of `HandledPromise.prototype` is
the same as the initial value of `Promise.prototype`.  Code that holds a
promise cannot sense whether it holds a handled or an unhandled promise.


### Handler traps

A handler object can provide handler traps (`get`, `set`, `delete`, `apply`, `applyMethod`) and their associated `*SendOnly` traps.

```ts
{
  get(target, prop): Promise<result>,
  getSendOnly(target, prop): void,
  set(target, prop, value): Promise<boolean>,
  setSendOnly(target, prop, value): void,
  delete(target, prop): Promise<boolean>,
  deleteSendOnly(target, prop): void,
  apply(target, args): Promise<result>,
  applySendOnly(target, args): void,
  applyMethod(target, prop, args): Promise<result>,
  applyMethodSendOnly(target, prop, args): void,
}
```

If the handler does not provide a `*SendOnly` trap, its default implementation
is the non-send-only trap with a return value of `undefined` (not a promise).

If the handler omits a non-send-only trap, invoking the associated operation
returns a promise rejection.  The only exception to that behaviour is if the
handler does not provide the `applyMethod` optimization trap.  Then, its
default implementation is
```js
HandledPromise.apply(HandledPromise.get(p, prop), args)
```

This expansion requires that the promise for the remote method be unnecessarily
reified.

For an unfulfilled handler, the trap's `target` argument is the unfulfilled
handled promise, so that it can gain control before the promise is resolved.
For a fulfilled handler, the method's `target` argument is the result of the
fulfillment, since it is available.

### `HandledPromise` static methods

The methods in this section are used to implement higher-level communication
primitives, such as the `E` proxy maker.

These methods are analogous to the `Reflect` API, but asynchronously invoke a
handled promise's handler regardless of whether the target has resolved.  This
is necessary in order to allow pipelining of messages before the exact
destination is known (i.e. after the handled promise is resolved).

```js
HandledPromise.get(target, prop); // Promise<result>
HandledPromise.getSendOnly(target, prop); // undefined
```

```js
HandledPromise.set(target, prop, value); // Promise<boolean>
HandledPromise.setSendOnly(target, prop value); // undefined
```

```js
HandledPromise.delete(target, prop); // Promise<boolean>
HandledPromise.deleteSendOnly(target, prop); // undefined
```

```js
HandledPromise.apply(target, [args]); // Promise<result>
HandledPromise.applySendOnly(target, [args]); // undefined
```

The `applyMethod` call combines property lookup with function application in
order to distinguish them from a `get` whose value is separately inspected, and
for the handler to be able to bundle the two operations as a single message.

```js
HandledPromise.applyMethod(target, prop, args); // Promise<result>
HandledPromise.applyMethodSendOnly(target, prop, args); // undefined
```

## Platform Support

All the above behavior, as described so far, is implemented in the [Eventual
Send Shim](https://github.com/Agoric/eventual-send) (TODO update to proposed
API).  However, there is one critical behavior that we specify, that can easily
be provided by a conforming platform, but is infeasible to emulate on top of
current platform promises.  Without it, many cases that should pipeline do not,
disrupting desired ordering guarantees. Consider:

```js
let pr;
const p = new Promise(r => pr = r);
E(p).foo();
let qr;
const q = new HandledPromise(r => qr = r, unresolvedHandler);
pr.resolve(q);
```

After `p` is resolved to `q`, the delayed `foo` invocation should be forwarded
to `q` and trap to `q`'s `unresolvedHandler`.  Although a shim could monkey
patch the `Promise` constructor to provide an altered `resolve` function which
does that, there are plenty of internal resolution steps that would bypass it.
There is no way for a shim to detect that unresolved unhandled promise `p` has
been resolved to unresolved handled `q` by one of these.  Instead, the `foo`
invocation will languish until a round trip fulfills `q`, thus
   * losing the benefits of promise pipelining,
   * arriving after messages that should have arrived after it.

## Syntactic Support

A separate [Wavy Dot Proposal](https://github.com/Agoric/proposal-wavy-dot)
proposes a more convenient syntax for calling the new internal methods proposed
here.  However, the eventual-send API described here is valuable even without
the wavy dot syntax.

## TODO

* Explain why we're choosing to modify Promise instead of using proxies.

* Explain how promise pipelining isn't just a fluent interface
