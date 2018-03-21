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
  throw when `deprecated` code is encountered or provide an embedder API to receive
  a callback when `deprecated` code is compiled, allowing embedders like Node.js the
  ability to custom handle it.
* VM implementations can automatically de-optimize code in any `deprecated` scope.
* Debuggers, IDEs, linters, and related tooling can provide visible warnings about
  use of `deprecated` code
