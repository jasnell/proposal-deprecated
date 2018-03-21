# deprecated reserved keyword

# Motivation

# Prior-art

# Proposal

The proposal is to introduce a new `deprecated;` keyword to the language such that
deprecated code can be easily identified in a standard way by tooling and the VM
implementations.

For example:

```js
function deprecatedFunction() {
  deprecated;
  // do stuff
}

function notDeprecatedFunction() {
  // do stuff
}
```

Alternatively, `deprecated;` can be a pragma type instruction:

```js
function deprecatedFunction() {
  'deprecated';
  // do stuff
}
```

Tooling support:

* VM implementations supporting the `deprecated` mechanism can provide an option to
  throw when `deprecated` code is encountered (similar in idea to `node --throw-deprecation`)
  or provide an embedder API to receive a callback when `deprecated` code is compiled, allowing
  embedders like Node.js the ability to custom handle it. (similar in idea to
  Node.js' `process.on('warning', (warning) => { /* ... */ })` API)
* VM implementations can automatically de-optimize code in any `deprecated` scope with
  an option to ignore the `deprecated` statement if necessary. The point is, there would
  be a penalty-by-default to use `deprecated` code that users would need to explicitly
  opt-out from. (similar in idea to: `node --no-deprecation`)
* Debuggers, IDEs, linters, and related tooling can provide visible warnings about
  use of `deprecated` code.
  
## Alternative: Using Decorators

The [Decorators](https://github.com/tc39/proposal-decorators) proposal offers a potential
alternative approach to this...
  
```js
@deprecated
function deprecatedFunction() {}
```

This approach, however, carries with it a number of important limitations:

1. The syntax is not backwards supportable or easily polyfillable. Specifically, the code would
   need to be transpiled to remove the decorator in order to run on older versions of Node.js and
   older browsers while the new keyword and pragma approach could be easily polyfilled.

2. Within Node.js, we often have need to deprecated *within* a function, for instance, in some
   cases only certain combinations of arguments to a function are deprecated. For the decorator
   method to work, then, we would need to be able to decorate individual blocks, e.g.,
   
```js
function foo(...args) {
  if (typeof args[0] === 'string') {
    // this is ok
  } else
  @deprecated {
    // this is deprecated  
  }
}
```

