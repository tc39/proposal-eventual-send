# ECMAScript Eventual Send: Support for distributed objects
By Mark S. Miller (@erights), Chip Morningstar (@FUDCo), and Michael FIG (@michaelfig)

**ECMAScript Eventual Send: Support for distributed objects**
## Summary

TODO

## Design Principles

1. Prevent plan interference between mutually untrusting peers by only allowing asynchronous communication.
2. Integrate with platform promises as the mechanism for asynchrony.
3. Allow for optimizations that reduce the cost of network latency.

## Details

TODO

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

All of the above `HandledPromise` static methods gracefully degrade if they are invoked on non-handled promises or non-promise values.  In doing so, the opportunity for network optimization is removed, but the basic functionality is consistent (e.g. still returning promises or undefined as expected).
