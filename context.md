# Observable Contexts

The core idea of this project is flattening observables, i.e. being able to treat observables as plain values within expressions. Imagine for a given observable, `a`, we use it within expressions as a plain value with the syntax `@a`. This means that for example if `a` is an observable of numeric values, then we can have
```js
@a + 2
```
Which should be an observable whose value changes whenever `a`'s value changes, and its value is always equal to that of `a` plus `2`:
```js
a.pipe(map(_a => _a + 2))
```

<br>

This document demonstrates why such syntax cannot be used without further constraints, as it will inevitably result in semantic contradictions. This issue can be solved by explicit specification of the scope/context in which such observable flattening can be allowed.

<br>

## The Problem

Given an arbitrary function, for example `console.log()`, how should the following statement behave?
```js
console.log(@a + 2)
```

By definition, for any given expression `F` (with at least one free variable), we expect the following codes to yield the same result:
```js
F(@a)
```
```js
a.pipe(map(_a => F(_a))
```
Assume `F` to be `console.log(x + 2)`, then we should have equivalance of the following codes:
```js
console.log(@a + 2)
```
```js
a.pipe(map(_a => console.log(_a + 2))
```
Which means our answer MUST BE _"an observable that, whenever `a` has a new value, logs that value plus `2`"_. We expect this to hold also if we rewrite the code as follows:
```js
const b = @a + 2
console.log(b)
```
However, by our own definition, this code should yield similar results to the following:
```js
const b = a.pipe(map(_a => _a + 2))
console.log(b)
```
Which means our answer MUST BE _"logs an observable object ONCE"_. Since this contradicts our previous result, it means that `@a` cannot be defined as described above, as otherwise we would have an inconsistent semantic for our syntax (or we would need to violate some key invariances, e.g. we shouldn't be to rewrite the code using variable `b` to begin with).

<br>

## The Solution

The only resolution would be to specify the intention explicitly, and have unspecified situations (which will be any out-of-context usage of `@a` construct) be syntax errors. For example, if we limit usage of `@a` to expressions that are denoted with `@ =>` (with same syntax hierarchy as anonymous functions), then this would be syntax error:
```js
console.log(@a + 2)
```
This would be the first option, i.e. _"an observable that, whenever `a` has a new value, logs that value plus `2`"_:
```js
@ => console.log(@a + 2)
```
And this would be the second option, i.e. _"logs an observable object ONCE"_:
```js
console.log(@ => @a + 2)
```

<br>

_💡 To summarize, for the general case, the scope/context in which observable values are treated as plain values in expressions MUST be explicitly specified, and out-of-context usage of any syntactic element enabling that behavior MUST be ruled as syntax error. In the rest of this document (and generally across this repository), we will call such contexts **Observable Contexts**._

<br>

## Consideration: JSX

It will be particularly useful to consider JSX expressions as observable contexts:

```jsx
render(<div>{@a + 2}</div>)
```

This is basically how libraries such as [SolidJS](https://www.solidjs.com) treat JSX expressions (in Solid lingo, observable context is called _tracking context_). However, that will violate equivalance of normal functions and JSX components, i.e. we won't be able to expect the following codes to be equal:
```jsx
<F>{x}</F>
```
```js
F({}, [x])

// or any other JSX component translation
```

<br>

Without assuming JSX expressions as observable contexts, we could still use the following construct:

```jsx
render(<div>{@ => @a + 2}</div>)
```

