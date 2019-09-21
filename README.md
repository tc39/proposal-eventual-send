# ECMAScript Eventual Send: Support for distributed objects
By Mark S. Miller (@erights), Chip Morningstar (@FUDCo), and Michael FIG (@michaelfig)

**ECMAScript Eventual Send: Support for distributed objects**
## Summary

TODO

## Design Principles

1. Prevent plan interference between mutually untrusting peers by only allowing asynchronous coupling.
2. Integrate with platform promises as the mechanism for asynchrony.
3. Allow for optimizations that reduce the cost of network latency.

## Details

TODO

### E and E.sendOnly Convenience Proxies

Probably the most common distributed programming case, invocation of remote methods with or without requiring return results, can be implemented by powerless proxies.  All authority needed to separate the peers can be implemented in the handled promise infrastructure.

The `E(target)` proxy maker wraps a remote target and allows for a single remote method call.  `E.sendOnly(target)` is similar, but declares that we do not use the result.

```js
import { E } from 'js:eventual-send';

// Invoke pipelined RPCs.
const fileP = E(
  E(target).openDirectory(dirName)
).openFile(fileName);
// Process the read results.
E(fileP).read().then(contents => {
  console.log('file contents', contents);
  E.sendOnly(fileP).write(contents + contents);
});
```

### HandledPromise constructor

```js
new HandledPromise(executor);
new HandledPromise(executor, unfulfilledHandler);
executor(resolveHandledPromise, reject);
resolveHandledPromise(presence, fulfilledHandler);
```

### HandledPromise static methods

These methods are analogous to the `Reflect` API, but asynchronously invoke a handled promise's handler (regardless of whether the target has resolved).  This is necessary in order to allow pipelining of messages before the exact destination is known (i.e. after the handled promise is resolved).

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
HandledPromise.apply(target, [args]);
HandledPromise.applySendOnly(target, [args]); // default: apply(foo, bar)
```

The `applyMethod` call combines property lookup with function application in order for the handler to be able to bundle the two operations as a single message.

```js
HandledPromise.applyMethod(target, prop, [args]); // default: get(prop).then(f => Reflect.apply(f, ...));
HandledPromise.applyMethodSendOnly(target, prop, [args]); // default: applyMethod()
```
