JavaScript
==========

## Node.js

- install
  - download `https://nodejs.org/dist/<ver>/node-<ver>-linux-x64.tar.xz`
  - untar to anywhere
  - add `$NODE/bin` to `PATH`
- install packages through npm
  - `npm install -g <package>`

## Resources

- <https://developer.mozilla.org/en/JavaScript/Guide>

## Data structures

- ECMAScript defines six data types
  - Number: double precision floating point
  - String
  - Boolean: true or false
  - Null: null
  - Undefined: undefined
  - Object: a bag of properties that can be added or removed
- All except objects are immutable
- Functions are objects that are callable
- Arrays are objects

## Inheritance and prototype chain

- An object has an internal link to another object (or null) called its
  prototype
  - this forms a prototype chain
- Own properties
  - consider this prototype chain
    `{a:1, b:2} ---> {b:3, c:4} ---> null`
  - the object has `a` and `b` own properties
  - the prototype has `b` and `c` own properties
  - `b` own property shadows prototype's `b` property
- setting a property to an object creates an own property
- Functions can be used as property values.  The only thing changed when
  functions are object properties is the value of `this` when the function is
  executed
- `var o = {a: 1};` has this prototype chain
  `o ---> Object.prototype ---> null`
- `var a = ["yo", "whadup", "?"]` has this prototype chain
  `a ---> Array.prototype ---> Object.prototype ---> null`
- `function f() { return 2; }` has this prototype chain
  `f ---> Function.prototype ---> Object.prototype ---> null`
- `function Graph() { this.foo = ...; } Graph.prototype = ...; var g = new Graph();`
  - `Graph` is a function
  - when called with `new`, it is known as a constructor
  - g is an object with own property `foo`
  - `g.[[Prototype]]` is the value of `Graph.prototype` at the time of
    initialization
- A new way to create an object is `Object.create`
  - `var a = {a: 1};` has `a ---> Object.prototype ---> null`
  - `var b = Object.create(a);` has `b ---> a ---> Object.prototype ---> null`
  - `var c = Object.create(b);` has `c ---> b ---> a ---> Object.prototype ---> null`
  - `var d = Object.create(null)` has `d ---> null`

## Ch. 3: Variables, literals

- `$` is a valid identifier character
- variables are declared with `var`
  - when there is no initializer, the value is `undefined`
- a variable declared outside of any function is a global variable
  - contrary, it is a local variable if declared within a function
  - there is no block-scoped variable
- variables are hoisted to the top of the function
- global variables are in fact properties of _the_ global object
- array literals: `[ elem1, elem2, ...]`
  - empty element means `undefined`
  - the last trailing comma is ignored
- `false` is not the same as a boolean object with value `false`
  - `var b = new Boolean(false)` is true in `if`-condition
- object literals: `{ prop1: val1, prop2: val2, ... }`
- `"` and `'` are the same for string literals

## Ch. 4: Expressions and operators

- four types of expressions
  - arithmetic expr evaluates to a number
  - string expr evaluates to a string
  - logical expr evaluates to a boolean
  - object expr evaluates to an object
- `delete` operator deletes an object, an object property, or an element in an
  array
- `in` returns true if the specified property is in the object
  - `3 in [2, 4, 6, 8]` is true
  - `"length" in [2, 4, 6, 8]` is true
- `instanceof` returns true if the specified object is of the specified type
  - `[1, 2] instanceof Array` is true
- `new` operator
- `this` keyword
- `typeof` operator returns the string representing the type
- `void` operator makes an expression not return any value

## Ch. 5: Regular expressions

- two ways to construct
  - `var re = /ab+c/g;`
  - `var re = new RegExp("/ab+c/", "g");`
- RegExp methods
  - `exec`
  - `test`
- String methods
  - `match`
  - `search`
  - `replace`
  - `split`
