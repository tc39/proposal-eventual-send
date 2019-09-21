# ECMAScript Eventual Send: Support distributed promise pipelining
By Mark S. Miller (@erights), Chip Morningstar (@FUDCo), and Michael FIG (@michaelfig)

**ECMAScript Eventual Send: Support distributed promise pipelining**
## Summary

Promises in Javascript were proposed in 2011 at the [ECMAScript strawman
concurrency
proposal](https://web.archive.org/web/20161026162206/http://wiki.ecmascript.org/doku.php?id=strawman:concurrency).
These promises descend from the [E language](http://erights.org/) via
the [Waterken Q library](http://waterken.sourceforge.net/web_send/)
and [Kris Kowal's Q library](https://github.com/kriskowal/q). A good
early presentation is Tom Van Cutsem's [Communicating Event Loops: An
exploration in Javascript](http://soft.vub.ac.be/~tvcutsem/talks/presentations/WGLD_CommEventLoops.pdf). Allof these are about promises as a first step towards distributed computing, by using promises as asynchronous references to remote objects.

Kris Kowal's [Q-connection
library](https://github.com/kriskowal/q-connection) extended Q's
promises for distributed computing with [promise pipelining](https://capnproto.org/rpc.html), essentially in the way we have in mind. However, in the absence of platform support for [Weak
References](https://github.com/tc39/proposal-weakrefs), this approach
was not practical. Given weak references, the [Midori
project](http://joeduffyblog.com/2015/11/19/asynchronous-everything/)
and [Cap'n Proto](https://capnproto.org/rpc.html), among others,
demonstrates that this approach to distributed computing works well at
scale.

We add *eventual-send* operations to promises, to express invocations of potentially remote objects. We explain the notion of a *handled Promise*, whose handler can provide alternate eventual-send behaviors. These mechanisms, together with weak references, enable writing remote object communications systems, but they are not specific to any one. This proposal does not include any specific usage of the mechanisms we propose, except as a motivating
example and test of adequacy.


## Design Principles

1. Prevent reentrancy attacks (a form of plan interference).
3. Support *promise pipelining* to reduce the cost of network latency.

## Details

To specify eventual-send operations and handled promises, we follow the pattern use to incorporate proxies into JavaScript: We specified...
   * internal methods that all objects must support.
   * static `Reflect` methods for invoking these internal methods.
   * invariants that these methods must uphold.
   * default behaviors of these methods for normal (non-exotic) objects.
   * how proxies implement these methods by delegating most of their behaviors to corresponding traps on their handlers.
   * the remaining behavior in the proxy methods to guarantee that these invariants are upheld despite arbitrary behavior by the handler.
   * fallback behaviors for absent traps, implemented in terms of the remaining traps.

Following this analogy, we add the internal eventual-send methods to all promises, provide default behaviors for unhandled promises, and introduce handled promises whose handlers provide traps for these methods.

We introduce a new constructor, `HandledPromise`, for making handled promises. The static methods below are static methods of this constructor.

| Internal Method | Static Method | Default Behaviour | Handler trap |
| --- | --- | --- |
| `p.[[GetSend]](prop)` | `get(p, prop)` | `p.then(t => t[prop])` | `h.get(t, prop)` |
| `p.[[SetSend]](prop, value)` | `set(p, prop, value)` | `p.then(t => (t[prop] = value))` | `h.set(t, prop, value)` |
| `p.[[DeleteSend]](prop)` | `delete(p, prop)` | `p.then(t => delete t[prop])` | `h.delete(t, prop)` |
| `p.[[ApplySend]](args)` | `apply(p, args)` | `p.then(t => t(...args))` | `h.apply(t, args)` |
| `p.[[ApplyMethodSend]](prop, args)`| `applyMethod(p, prop, args)` | `p.then(t => t[prop](...args))` | `h.applyMethod(t, prop, args)` |

To protect against reentrancy, the proxy internal method postpones the execution of the handler trap to a later turn, and immediately returns a promise for what the trap will return. For example, for the [[GetSend]] internal method of handled promises is effectively

```js
p.then(t => h.get(t, prop))
```

Sometimes, these operations will be used to cause remote effects while ignoring the local promise for the result. For distributed messaging protocols, the extra bookkeeping for these return results are sufficiently expensive that we should be able to avoid it when unneeded. To support this, we introduce the "SendOnly" variants of these methods. We show only the SendOnly variant of the [[Get]] trap, as all the others follow exactly the same pattern.

| Internal Method | Static Method | Default Behaviour | Handler trap |
| --- | --- | --- |
| `p.[[GetSendOnly]](prop)` | `getSendOnly(p, prop)` | `void p.then(t => t[prop])` | `h.getSendOnly(t, prop)` |

No matter what the SendOnly handler trap returns, the proxy internal [[\*SendOnly]] method always immediately returns `undefined`.

When a "SendOnly" trap is absent, the trap behavior defaults to the corresponding non-SendOnly trap. But again, the proxy internal [[\*SendOnly]] method always immediately returns `undefined`, and so is effectively

```js
void p.then(t => h.get(t, prop))
```

### E and E.sendOnly Convenience Proxies

Probably the most common distributed programming case, invocation of remote methods with or without requiring return results, can be implemented by powerless proxies.  All authority needed to enable communication between the peers can be implemented in the handled promise infrastructure.

The `E(target)` proxy maker wraps a remote target and allows for a single remote method call returning a promise for the result.

```js
E(target).method(arg1, arg2...) // Promise<result>
```

`E.sendOnly(target)` is similar, but declares that we do not want the result (or even acknowledgement).

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

### HandledPromise constructor

In a manner analogous to *Proxy* handlers, a **handled promise** is associated with a handler object.

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

This handler is not exposed to the user of the handled promise, so it provides a secure separation between the unprivileged client (which uses the `E`, `E.sendOnly` or static `HandledPromise` methods) and the privileged  system which implements the communication mechanism.

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

If the handler does not provide a `*SendOnly` trap, its default implementation is the non-send-only trap with a return value of `undefined` (not a promise).

If the handler omits a non-send-only trap, invoking the associated operation returns a promise rejection.  The only exception to that behaviour is if the handler does not provide the `applyMethod` optimization trap.  Then, its default implementation is `HandledPromise.get(target, prop).then(fn => fn(...args))`, which typically requires a round trip before `fn` can be resolved.

For an unfulfilled handler, the trap's `target` argument is the unfulfilled handled promise, so that it can gain control before the promise is resolved.  For a fulfilled handler, the method's `target` argument is the result of the fulfillment, since it is available.

### HandledPromise static methods

The methods in this section are used to implement higher-level communication primitives, such as the `E` proxy maker.

These methods are analogous to the `Reflect` API, but asynchronously invoke a handled promise's handler regardless of whether the target has resolved.  This is necessary in order to allow pipelining of messages before the exact destination is known (i.e. after the handled promise is resolved).

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

The `applyMethod` call combines property lookup with function application in order to distinguish them from a `get` whose value is separately inspected, and for the handler to be able to bundle the two operations as a single message.

```js
HandledPromise.applyMethod(target, prop, args); // Promise<result>
HandledPromise.applyMethodSendOnly(target, prop, args); // undefined
```

## Platform Support

All the above behavior, as described so far, is implemented in the *eventual-send-shim*. However, there is one critical behavior that we specify, that can easily be provided by a conforming platform, but is infeasible to emulate on top of current platform promises. Without it, many cases that should pipeline do not, disrupting desired ordering guarantees. Consider:

```js
let pr;
const p = new Promise(r => pr = r);
E(p).foo();
let qr;
const q = new HandledPromise(r => qr = r, unresolvedHandler);
pr.resolve(qr);
```

After `p` is resolved to `q`, the delayed `foo` invocation should be forwarded to `q` and trap to `q`'s `unresolvedHandler`. Although a shim could monkey patch the `Promise` constructor to provide an altered `resolve` function which does that, there are plenty of internal resolution steps that would bypass it. There is no way for a shim to detect that unresolved unhandled promise `p` has been resolved to unresolved unhandled `q` by one of these.
