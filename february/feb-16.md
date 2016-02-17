Day 3 - Invariant Violations
===================================


Even if you've only been using React for a few hours, you've almost certainly seen a console error pop up, describing something called an "invariant violation".

I've been using React for quite some time now, but I'm still not entirely sure what an invariant violation is!

Searching for "react invariant violation" will reveal tons of Stack Overflow questions about specific instances, but nowhere will you find a detailed technical explanation of invariant violations, how they work, what their philosophy is.

Thankfully, we have the source code! Let's see what we can figure out.

------------------

It turns out, the logic for this file lives in one very tiny file, `invariant.js`.

The file begins with an explanation:

```js
/**
 * Use invariant() to assert state which your program assumes to be true.
 *
 * Provide sprintf-style format (only %s is supported) and arguments
 * to provide information about what broke and what you were
 * expecting.
 *
 * The invariant message will be stripped in production, but the invariant
 * will remain to ensure logic does not differ in production.
 */

```

So right away, something becomes clear: `invariant violations` aren't a specific type of exception, they're a wrapper tool for throwing exceptions when assertions fail. This way, as you write your code, you can quickly write assertions, and if they fail, the program's execution stops, and an

There is a single `invariant` function, and it takes 8 arguments:

```js
function invariant(condition, format, a, b, c, d, e, f) { }
```

The first argument is easy to work out; It's an expression, and its use is to determine whether an invariant violation has occurred. If it's truthy, everything's cool; nothing is thrown.

If it's falsy, we have a problem. Here's that block of code:

```js
if (!condition) {
  var error;
  if (format === undefined) {
    error = new Error('Minified exception occurred; use the non-minified dev environment ' + 'for the full error message and additional helpful warnings.');
  } else {
    var args = [a, b, c, d, e, f];
    var argIndex = 0;
    error = new Error(format.replace(/%s/g, function () {
      return args[argIndex++];
    }));
    error.name = 'Invariant Violation';
  }

  error.framesToPop = 1; // we don't care about invariant's own frame
  throw error;
}

```

There's something interesting and unconventional happening here; a function is being passed to `String.prototype.replace`.

the regular expression is global (appended with /g), so that means this function will run on _every_ match. And, because we set up an ever-incrementing `argsIndex`, it means we can replace placeholders in the `format` string with up to 6 pre-defined values.

Here's an example, taken from `ReactElementValidator` (I've taken the liberty of cleaning it up a bit and indenting it, for clarity):

```js
invariant(
  false,
  '%s: %s type `%s` is invalid; it must be a function, usually from React.PropTypes.',
  componentName || 'React class',
  ReactPropTypeLocationNames[location],
  propName
)
```

Our first argument is `false`, which is (obviously) a falsy value.

The second argument is a string, with three occurrences of `%s`.

The next 3 arguments are the variables to fill into those placeholders.

---------

This was delightfully to-the-point. Now I know what invariant violations are!
