Day 5 - Dipping into setState
===================================


Over the past 4 days, we've skirted around looking at interesting but relatively insignificant portions of the React codebase. This is by design, and will likely continue for at least a few more days, because the core code is likely to be sufficiently massive that we'll want to know as much about the smaller bits as possible beforehand.

That said, I can't resist dipping my toes into the meat of the codebase. Or, er, choose your own less-disgusting metaphor.

Today we're looking at how state gets updated in `ReactComponent.js`

----------------

The `setState` method is very concise:

```js
ReactComponent.prototype.setState = function (partialState, callback) {
  !(typeof partialState === 'object' || typeof partialState === 'function' || partialState == null) ? process.env.NODE_ENV !== 'production' ? invariant(false, 'setState(...): takes an object of state variables to update or a ' + 'function which returns an object of state variables.') : invariant(false) : undefined;
  if (process.env.NODE_ENV !== 'production') {
    process.env.NODE_ENV !== 'production' ? warning(partialState != null, 'setState(...): You passed an undefined or null state object; ' + 'instead, use forceUpdate().') : undefined;
  }
  this.updater.enqueueSetState(this, partialState);
  if (callback) {
    this.updater.enqueueCallback(this, callback);
  }
};

```

We have a couple of checks to make sure we're supplying `setState` with valid input, and then we queue up our state change. If we've provided a callback, we queue it up as well.

There's apparently an `updater` object on components created by ReactComponent, and it holds a method to set state. Let's see if we can learn more about it.

The constructor for ReactComponent looks like this:

```js
function ReactComponent(props, context, updater) {
  this.props = props;
  this.context = context;
  this.refs = emptyObject;
  // We initialize the default updater but the real one gets injected by the
  // renderer.
  this.updater = updater || ReactNoopUpdateQueue;
}
```

So, clearly there's a rabbit-hole to go down here with figuring out what the real updater is. For now, though, the default should be good enough to answer some of our questions (hopefully).

I did a project-search for `enqueueSetState`, and found a file called `ReactUpdateQueue.js`.

Here's what that looks like:

```js
/**
 * Sets a subset of the state. This only exists because _pendingState is
 * internal. This provides a merging strategy that is not available to deep
 * properties which is confusing. TODO: Expose pendingState or don't use it
 * during the merge.
 *
 * @param {ReactClass} publicInstance The instance that should rerender.
 * @param {object} partialState Next partial state to be merged with state.
 * @internal
 */
enqueueSetState: function (publicInstance, partialState) {
  var internalInstance = getInternalInstanceReadyForUpdate(publicInstance, 'setState');

  if (!internalInstance) {
    return;
  }

  var queue = internalInstance._pendingStateQueue || (internalInstance._pendingStateQueue = []);
  queue.push(partialState);

  enqueueUpdate(internalInstance);
}
```

This is kinda what I feared; there are a bunch of things here that I know nothing about:

- publicInstance
- getInternalInstanceReadyForUpdate
- internalInstance.\_pendingStateQueue
- enqueueUpdate

This may seem like a disappointing discovery, but it's actually good. We now have a bunch of directions to explore. Much like how ReactElement recursively unpacks Components into DOM nodes until it has a complete understanding of the document, we'll dig down into each of these ideas and assemble them into a complete picture.

(bit of a throwback to yesterday's discoveries :) )
