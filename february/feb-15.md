Day 2 - Parents vs. Owners
===================================


Yesterday, I stumbled upon two concepts that seemed synonymous to me: `parents` and `owners`.

Rather than start by trying to find this distinction in the code, I did some googling, and found this key bit of data:

    In React each child has both a "parent" and an “owner”. The owner is the component that created a ReactElement. I.e. the render method which contains the JSX or createElement callsite.

    ```js
    class Foo {
      render() {
        return <div><span /></div>;
      }
    }
    ```

    In this example, the owner of the span is Foo but the parent is the div.


Link: https://facebook.github.io/react/blog/2015/02/24/streamlining-react-elements.html#owner

Instantly, so much of yesterday makes more sense. The `owner` is always the component responsible for creating this reactElement. The `parent` is just the enclosing element; it could be the owner, but it could also be any other element the component creates.

It also turns out that `owners` are much less significant in React >0.13. They've switched context to depend on parents, and if you use the new function-style of `refs`, the fact that refs use owners won't come up.

That said, I'm still curious to see how owners are established. Yesterday, I saw the mysterious property `ReactCurrentOwner.current`, and it appears in the main `ReactElement.js` file as well.

I opened the `ReactCurrentOwner.js` file and gasped. It is a single statement:

```js
/**
 * Keeps track of the current owner.
 *
 * The current owner is the component who should own any components that are
 * currently being constructed.
 */
var ReactCurrentOwner = {

  /**
   * @internal
   * @type {ReactComponent}
   */
  current: null

};

module.exports = ReactCurrentOwner;
```

That's it! That's the whole module!

How can this be? It's not a constructor, or something concatenated for every element. This library is imported, and the property checked, when creating and cloning elements!

I did a quick project search for `ReactCurrentOwner`, specifically trying to see where `.current` gets set. As far as I can tell, it is only ever set in two files: `ReactCompositeComponent`, and `ReactMultiChild`.

## Composite Components

It turns out there's a fair bit of misinformation on what composite components really are; the [TestUtils documentation](https://facebook.github.io/react/docs/test-utils.html#iscompositecomponent) imply that composite components are just ones created using React.createClass (and not, say, functional stateless components).

The true answer, though, is that composite components are components which are themselves comprised of components. A component composed of components. What a combo.

#### Back to the setting of `ReactCurrentOwner.current`

It turns out, `ReactCurrentOwner.current` is only set in two situations, in ReactCompositeComponent.js: In development mode, and when measuring performance with ReactPerf.

In development mode, we see this:

```js
if (process.env.NODE_ENV !== 'production') {
  ReactCurrentOwner.current = this;
  try {
    inst = new Component(publicProps, publicContext, ReactUpdateQueue);
  } finally {
    ReactCurrentOwner.current = null;
  }
```

`finally` is an option in try/catch blocks that runs regardless of whether there were any errors or not.

So, we set `ReactCurrentOwner.current` to the current composite component, create a new component instance, and then immediately unset `ReactCurrentOwner.current`

This way, the component instance is created when ReactCurrentOwner is set to the current component. This current component becomes the _owner_ of this brand new component instance. Ah-ha.

This seems like an odd strategy, but really all we're doing is passing a piece of information between modules. Because we take care to immediately unset it _even if the code crashes_, we can be safe in our knowledge that `ReactCurrentOwner.current` either holds the new component's owner, or `null`.

Lightning bulb moment achieved, it's time for bed.
