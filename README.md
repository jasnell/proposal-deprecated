# Deprecated Stuff

# Motivation

# Prior-art

# Proposal

The proposal is to introduce a new `deprecated;` global to the language such that
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

## Polyfills and Dealing with Ambiguity

To make this easily polyfillable and to avoid breaking existing code, `deprecated`
here can actually be a global with a very specific value as opposed to a language
keyword. For instance, a new `Symbol.deprecated` well-known standard symbol can
be defined with `global.deprecated = Symbol.deprecated` by default. Then, when
the VM or tooling encounters `deprecated;` in the code, and `deprecated` is equal
to `Symbol.deprecated`, it is handled as described.

On older VMs that do not understand the special meaning of `deprecated;` when it has
been polyfilled, the statement would be a non-op.

VMs should have little difficulty in determining wether or not to de-opt at runtime
but tooling can see some ambiguity here.. for instance:

```js
function foo(deprecated) {
  deprecated;  // Only triggers deprecated semantics in the VM if deprecated === Symbol.deprecated
               // Causes ambiguity in tooling because it cannot determine reliably if deprecated === Symbol.deprecated
}
```

Note: the VM behavior expected here would be an automatic de-opt of the code by default,
and possibly a throw if a given flag was enabled.

In this case, the best thing the tooling could do is show a warning about the
ambiguous use and provide a mechanism (such as a lint ignore rule) to opt-out
of the checking for that particular case.

Another ambiguous case would be what happens if a user assigns `Symbol.deprecated`
to a different variable. Would it have the same effect? The answer is no. The
explicit requirement is an expression that is exactly `deprecated;` (with or without
the `;` but that's a different argument entirely).

```js
function foo() {
  const m = deprecated; // does not trigger the 'deprecated' semantics
  m; // does not trigger the 'deprecated' semantics
  
  deprecated; // does trigger the 'deprecated' semantics
  deprecated  // does trigger the 'deprecated' semantics
}

function foo() {
  deprecated = 'something else';
  deprecated; // does not trigger the 'deprecated' semantics
}
```

We *could* go with an approach like `import { deprecated } from 'something'` to pull in the intrinsic rather
than making it global. 

Specifically,

```js
import { deprecated as dep } from 'something'

function foo() {
  dep;
 }
```

This would be fine from a VM behavior perspective but makes things more difficult
from a tooling perspective specifically because the tooling would have to perform
additional more complex analysis to know what `dep;` actualy means.

Tools like linters may consistently struggle with reliably detecting use of deprecated
code even using a mechanism such as this. For instance,

```js
class Foo {
  bar() {
    deprecated
  }
}

require('some-module-that-monkey-patches-Foo')

var foo = new Foo()
foo.bar()
```

In this case, tools would have to be able to reliably determine if `bar()` has
been monkeypatched in order to give any kind of reasonable warning. This is why
relying strictly on tools like linting is not sufficient. There's no silver bullet
around resolving this particular ambiguity.

## Pragma option: `'deprecated';`

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
   older browsers while the new global and pragma approach could be easily polyfilled.

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

## Alternative: `deprecated()` Intrinsic

In Node.js, we do not only mark code as being deprecated, we also associate a static identifier and message with the deprecation to provide helpful additional information to the user. While this has proven to have mixed results, we could follow the same basic pattern here while preserving the semantics discussed above.

```js
function deprecatedFunction() {
  deprecated('Use something else');
  // so some stuff
}
```

The default VM behavior would be the same: automatic de-opt when encountered with an option of throwing
if a given flag is used. The message given to the function would provide the error message text.

This approach is not mutually exclusive with the proposed approach in that rather than defining
`global.deprecated === Symbol.deprecated()`, we just make `global.deprecated === [Intrinsic function deprecated()]`,
with the same fundamental semantics.

Where this could run in to issues, however, is in the ambiguous case described earlier where `deprecated` is
assigned some other value by user code:

```js
function foo(deprecated) {
  deprecated('Use something else') // fails if deprecated happens to not be a function
}
```
