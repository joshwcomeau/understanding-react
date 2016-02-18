Day 4 - React.createElement
===================================


Earlier today, I was reading through the React guides, and found this article on [the differences between components, elements and instances in React](https://facebook.github.io/react/blog/2015/12/18/react-components-elements-and-instances.html).

It was truly illustrative; it tied together a bunch of stuff I kinda-sorta-half-knew, and made me feel like I had a much better handle on what createElement does.

As a refresher, React elements are what JSX compiles to, and they look like:

```js
{
  type: Avatar,
  props: {
    className: 'tall blue',
    children: {
      type: 'p',
      props: {
        children: 'Hello there!'
      }
    }
  }
}
```

Elements can represent native DOM nodes, or custom components. When an element's _type_ is a class or function, it assumes its a custom component, and will 'resolve' it into its constituent parts, until all elements represent native DOM nodes.

This collection of elements that represent DOM nodes is what makes up the virtual DOM.

JSX will compile to calls to React.createElement, which is a helper used to create these objects.


Now that I've gotten a pretty good handle on the theory, let's see how it looks in the source :)

------------

`ReactElement.js` is surprisingly small; 200-some lines with comments and sanity checks.


It includes a base constructor, and the comments tell us that it's only there to allow `instanceof` checks; this is misleading though. The constructor code actually does get run, and affects the created elements!

It assigns `$$typeof`, a property that just lets us know that this object is a ReactElement, and then freezes both its props and itself:

```js
Object.freeze(element.props);
Object.freeze(element);
```

`Object.freeze` is a way to make mutable javascript objects _immutable_. ReactElements are always immutable, and I suspect it's to make it very easy to compare elements when React does its diffing reconciliation thing.



The primary `ReactElement.createElement` method is remarkably concise:

```js
var RESERVED_PROPS = {
  key: true,
  ref: true,
  __self: true,
  __source: true
};

ReactElement.createElement = function (type, config, children) {
  var propName;

  // Reserved names are extracted
  var props = {};

  var key = null;
  var ref = null;
  var self = null;
  var source = null;

  if (config != null) {
    ref = config.ref === undefined ? null : config.ref;
    key = config.key === undefined ? null : '' + config.key;
    self = config.__self === undefined ? null : config.__self;
    source = config.__source === undefined ? null : config.__source;
    // Remaining properties are added to a new props object
    for (propName in config) {
      if (config.hasOwnProperty(propName) && !RESERVED_PROPS.hasOwnProperty(propName)) {
        props[propName] = config[propName];
      }
    }
  }

  // Children can be more than one argument, and those are transferred onto
  // the newly allocated props object.
  var childrenLength = arguments.length - 2;
  if (childrenLength === 1) {
    props.children = children;
  } else if (childrenLength > 1) {
    var childArray = Array(childrenLength);
    for (var i = 0; i < childrenLength; i++) {
      childArray[i] = arguments[i + 2];
    }
    props.children = childArray;
  }

  // Resolve default props
  if (type && type.defaultProps) {
    var defaultProps = type.defaultProps;
    for (propName in defaultProps) {
      if (typeof props[propName] === 'undefined') {
        props[propName] = defaultProps[propName];
      }
    }
  }

  return ReactElement(type, key, ref, self, source, ReactCurrentOwner.current, props);
};

```

Our first order of business is to pluck data out of `config` - we set a few special variables (key, ref, self and source) based on the content of config, and assign any other properties to a `props` object.

Children can be a single object, or an array of multiple objects. We start by getting the length of the children by taking the number of arguments and subtracting two (since the first two arguments are `type` and `config`).

If we have exactly 1 remaining argument, we have a single child, and we can just assign `props.children` to it directly.

If we have more than 1 remaining argument, we make an array of the appropriate length, and push our arguments in, in order. Then, we set `props.children` to this array.

Finally, we set our `defaultProps`, and return this newly created element!


-------

This whole process is delightfully simple and straight-forward; ReactElements are so simple!




...there's a mystery left in the constructor, though.

we attach a property, `_store`, to our element, in the constructor. We define non-enumerable properties `validated`, `_self`, and `source` to it. The latter two are development-only (according to a comment).

Even though we freeze `element`, we never freeze `element._store`, so it remains mutable.

Something to investigate another time, though!
