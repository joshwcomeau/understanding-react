Day 1 - Initial setup and thoughts
===================================


React's repo is massive, and incredibly confusing. Rather than explore on github, I add it to the package.json for this file and `npm install` it. The /lib/ directory has all the core React code, in a flat directory structure. Perfect.

After a few minutes of digging, it looks like what I would consider the "main" file: `/src/isomorphic/ReactIsomorphic.js`.

It imports all of the main modules one would expect; ReactComponent, ReactElement, ReactClass, etc.

```js
/* ReactIsomorphic.js */
var ReactChildren = require('./ReactChildren');
var ReactComponent = require('./ReactComponent');
var ReactClass = require('./ReactClass');
var ReactDOMFactories = require('./ReactDOMFactories');
var ReactElement = require('./ReactElement');
var ReactElementValidator = require('./ReactElementValidator');
var ReactPropTypes = require('./ReactPropTypes');
var ReactVersion = require('./ReactVersion');

```


Within a few lines, we see our first peek at how React changes its behaviour based on `process.env.NODE_ENV`:

```js
var createElement = ReactElement.createElement;
var createFactory = ReactElement.createFactory;
var cloneElement = ReactElement.cloneElement;

if (process.env.NODE_ENV !== 'production') {
  createElement = ReactElementValidator.createElement;
  createFactory = ReactElementValidator.createFactory;
  cloneElement = ReactElementValidator.cloneElement;
}
```


It seems pretty obvious, by the naming convention, that the development versions of these core React methods simply add some validations. Let's take a closer look at validator.

Aside from _maybe_ PropType warnings, the React error I see the most is when I forget to assign a `key` to a dynamically-generated array of elements. It looks like ReactElementValidator's first order of business is validating the presence of `keys` on dynamically-generated arrays of children

(sidenote: I recently read Facebook's [article on reconciliation](https://facebook.github.io/react/docs/reconciliation.html), which shed a whole bunch of light on React's inner workings, including why keys are necessary to keep track of arrays of elements).

Pretty quickly, I come upon this:

```js

/**
 * Warn if there's no key explicitly set on dynamic arrays of children or
 * object keys are not valid. This allows us to keep track of children between
 * updates.
 */
var ownerHasKeyUseWarning = {};

var loggedTypeFailures = {};

/**
 * Warn if the element doesn't have an explicit key assigned to it.
 * This element is in an array. The array could grow and shrink or be
 * reordered. All children that haven't already been validated are required to
 * have a "key" property assigned to it.
 *
 * @internal
 * @param {ReactElement} element Element that requires a key.
 * @param {*} parentType element's parent's type.
 */
function validateExplicitKey(element, parentType) {
  if (!element._store || element._store.validated || element.key != null) {
    return;
  }
  element._store.validated = true;

  var addenda = getAddendaForKeyUse('uniqueKey', element, parentType);
  if (addenda === null) {
    // we already showed the warning
    return;
  }
  process.env.NODE_ENV !== 'production' ? warning(false, 'Each child in an array or iterator should have a unique "key" prop.' + '%s%s%s', addenda.parentOrOwner || '', addenda.childOwner || '', addenda.url || '') : undefined;
}

/**
 * Shared warning and monitoring code for the key warnings.
 *
 * @internal
 * @param {string} messageType A key used for de-duping warnings.
 * @param {ReactElement} element Component that requires a key.
 * @param {*} parentType element's parent's type.
 * @returns {?object} A set of addenda to use in the warning message, or null
 * if the warning has already been shown before (and shouldn't be shown again).
 */
function getAddendaForKeyUse(messageType, element, parentType) {
  var addendum = getDeclarationErrorAddendum();
  if (!addendum) {
    var parentName = typeof parentType === 'string' ? parentType : parentType.displayName || parentType.name;
    if (parentName) {
      addendum = ' Check the top-level render call using <' + parentName + '>.';
    }
  }

  var memoizer = ownerHasKeyUseWarning[messageType] || (ownerHasKeyUseWarning[messageType] = {});
  if (memoizer[addendum]) {
    return null;
  }
  memoizer[addendum] = true;

  var addenda = {
    parentOrOwner: addendum,
    url: ' See https://fb.me/react-warning-keys for more information.',
    childOwner: null
  };

  // Usually the current owner is the offender, but if it accepts children as a
  // property, it may be the creator of the child that's responsible for
  // assigning it a key.
  if (element && element._owner && element._owner !== ReactCurrentOwner.current) {
    // Give the component that originally created this child.
    addenda.childOwner = ' It was passed a child from ' + element._owner.getName() + '.';
  }

  return addenda;
}
```

This is a huge chunk, but it looks like it's all dealing with the same premise; catching and alerting when the children are lacking keys.

To tell if the element has a valid key, it invokes the `validateExplicitKey` function. It takes the element and the type of the parent as arguments, and it has 3 main parts:

  1. Initial validations
  Figuring out if we can quit early, if the element already has a key or has been validated in a previous iteration.

  2. Generating Addenda

  3. Trigger the warning


1. Initial validations

The first line in the function is:

```js
if (!element._store || element._store.validated || element.key != null) {
  return;
}

element._store.validated = true;
```

I'm not entirely sure what \_store is, but the final part of that condition is a giveaway; if the element's key is null (or undefined, because it's using the loose operator `==`), the element has a key, therefore it's valid, so we can return.

The second statement, setting \_store.validated to true, is another big giveaway about the first step. We set the element to valid now, so that if we see it again in a future run, we'll know that it's already been through this process.

(Will need to come back and figure out what the deal is with \_store, though)

2. Generating Addenda

Warning messages are serious business, and we need to make sure we provide helpful information. This function makes a call to `getDeclarationErrorAddendum`, which looks like:

```js
function getDeclarationErrorAddendum() {
  if (ReactCurrentOwner.current) {
    var name = ReactCurrentOwner.current.getName();
    if (name) {
      return ' Check the render method of `' + name + '`.';
    }
  }
  return '';
}
```

ReactCurrentOwner is something I've seen a bit about, in various open-source libraries, but I'm still not clear on its duties. In this case, we're just looking to figure out which component "owns" this element, so that when we throw a warning, we can point the developer in the right direction. That said, I have no idea what ReactCurrentOwner.current is, or where that gets set.

If we don't have the current owner, we return an empty string, which causes this conditional block to be executed:

```js
if (!addendum) {
  var parentName = typeof parentType === 'string' ? parentType : parentType.displayName || parentType.name;
  if (parentName) {
    addendum = ' Check the top-level render call using <' + parentName + '>.';
  }
}
```

If ReactCurrentOwner.current fails, we rely instead on `parentType`, a variable passed in from `validateExplicitKey`. Not sure where this comes from, as I haven't seen it in action, but it's pretty clear that we're just trying to figure out where to point the user to find the key-less child.

Next up, we come upon a neat little JS trick:

```js
var memoizer = ownerHasKeyUseWarning[messageType] || (ownerHasKeyUseWarning[messageType] = {});
if (memoizer[addendum]) {
  return null;
}
memoizer[addendum] = true;
```

The first line is complicated, but elegant. On the left side of the || you have a lookup on an object created at the top of the file. THe first time this file runs, that object is empty, and so it will definitely fail.

If it fails, it moves to the right side of the ||, where it assigns an empty object to that position in the object. Assignment returns the value being assigned, which means that both `memoizer` and `ownerHasKeyUseWarning[messageType]` both point to the same new empty object.

If the memoizer has already seen this addendum, we're done. Otherwise, we set it as a property on the memoizer, so that future iterations will return at this point.

If we have not already seen this addendum, we create the addenda:

```js
var addenda = {
  parentOrOwner: addendum,
  url: ' See https://fb.me/react-warning-keys for more information.',
  childOwner: null
};
```

Here we illustrate (but don't really explain) my earlier confusion about parent Vs. Owner: Clearly they aren't the same thing! `addendum` at this point will either be an owner-specific message or a parent-specific mesage, though, so this key seems apt.

Finally, one last check:

```js
// Usually the current owner is the offender, but if it accepts children as a
// property, it may be the creator of the child that's responsible for
// assigning it a key.
if (element && element._owner && element._owner !== ReactCurrentOwner.current) {
  // Give the component that originally created this child.
  addenda.childOwner = ' It was passed a child from ' + element._owner.getName() + '.';
}
```

This is pretty straightforward. If the owner accepts children, and the children are what break, the owner of the children might be the culprit.

Ahh. I think I'm starting to see the distinction between owners and parents. Sorta. Not really.

3. Trigger the warning

After all that, we have an `addenda`, and it's ready to be displayed!

This last bit just creates a long string out of the various addenda fields, and passes it on to the `warning` method, assuming we're in production.


-----------------------------------

Awesome. We've only scratched the surface of one of the most inconsequential portions of the React codebase, but we've accomplished something far greater: we've gotten our feet wet, and discovered that React is well-commented, well-laid-out (once you can find the main file), and fairly intuitive.

I've already seen a couple of neat tricks, and I can't wait to see where it takes me (=
