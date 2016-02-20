Day 6 - Further down the SetState rabbit hole
===================================

Yesterday, we discovered that to understand React's setState, we'd need a lot of precursory knowledge. Let's try and flesh some of that knowledge out!

ReactUpdateQueue.js is where the `enqueueSetState` method lives, and its first order of business is to invoke this method:

```js
var internalInstance = getInternalInstanceReadyForUpdate(publicInstance, 'setState');
```

That method is defined at the top of the file. With the validity checks removed, it is simply:

```js
function getInternalInstanceReadyForUpdate(publicInstance, callerName) {
  var internalInstance = ReactInstanceMap.get(publicInstance);
  if (!internalInstance) {
    return null;
  }

  return internalInstance;
}
```

Let's keep digging, and see what ReactInstanceMap is:

```js
/**
 * `ReactInstanceMap` maintains a mapping from a public facing stateful
 * instance (key) and the internal representation (value). This allows public
 * methods to accept the user facing instance as an argument and map them back
 * to internal methods.
 */

var ReactInstanceMap = {

  /**
   * This API should be called `delete` but we'd have to make sure to always
   * transform these to strings for IE support. When this transform is fully
   * supported we can rename it.
   */
  remove: function(key) {
    key._reactInternalInstance = undefined;
  },

  get: function(key) {
    return key._reactInternalInstance;
  },

  has: function(key) {
    return key._reactInternalInstance !== undefined;
  },

  set: function(key, value) {
    key._reactInternalInstance = value;
  },

};
```

What it's doing is pretty clear, but what it _means_ isn't. We have these convenience getters and setters for fetching the internal version of a public object, but I'm still not sure what `publicInstance` is (or how it differs from `internalInstance`).

Rewinding to where we started yesterday, in `ReactComponent.prototype.setState`, `publicInstance` is just the component itself. It's the object whose prototype holds the `setState` method.

Let's see if we can see where ReactInstanceMap.set gets called.

It's only called once in all of React, in `ReactCompositeComponent.js`, in `mountComponent`.

`mountComponent` looks truly fascinating, and like it holds a ton of significant logic. It also looks like, in the short term, it would create more questions than answers.

Let's come back to this another time.

In `ReactUpdateQueue`, we get our internal instance ready for update, which really just involves using the ReactInstanceMap getter to nab and return the internal instance.

Then we hit this:

```js
var queue =
  internalInstance._pendingStateQueue ||
  (internalInstance._pendingStateQueue = []);

queue.push(partialState);
```

So we start by fetching \_pendingStateQueue, or assigning it to an empty array if there is no queue. Then, we push our new partial state into the queue.

Finally, the last line:

```js
enqueueUpdate(internalInstance);
```

`enqueueUpdate` is defined earlier in the file. It's this:

```js
function enqueueUpdate(internalInstance) {
  ReactUpdates.enqueueUpdate(internalInstance);
}
```

Why this wrapper function is necessary is beyond me (couldn't we just have called ReactUpdates.enqueueUpdate directly? Or made `enqueueUpdate` a variable referencing ReactUpdates.enqueueUpdate?), but it defers to yet another file.

Clearly, then, the job of ReactUpdateQueue is simply to prep and track updates before delegating to ReactUpdates.


----------------

ReactUpdates looks awesome; a quick pass-through reveals a bunch of real javascript (instead of a bunch of validity checks and delegation), as well as some interesting new concepts (I wonder what a batching strategy is?!)

Alas, it's after 10pm on a Friday, my joints hurt, and a hot cup of tea sounds delightful. God I'm old.
