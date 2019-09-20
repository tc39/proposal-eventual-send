```js
new HandledPromise(executor, [unfulfilledHandler]);
HandledPromise.get();
HandledPromise.set();
HandledPromise.delete();
HandledPromise.apply();
HandledPromise.applyMethod(); // default: get(prop).then(f => Reflect.apply(f, ...));
HandledPromise.getOneWay(); // default: get(prop)
HandledPromise.setOneWay(); // default: set(prop)
HandledPromise.deleteOneWay(); // default: delete(prop)
HandledPromise.applyOneWay(); // default: apply(foo, bar)
HandledPromise.applyMethodOneWay(); // default: applyMethod()
E(x);
E.oneWay(x);
```
