# Core Idea

The core idea of the proposed solution is syntactically _flattening_ observable values, i.e. making values wrapped inside an observable accessible to use within expressions, similar to how `await` keyword _flattens_ `Promise`s. Much like flattening `Promise`s, flattening observables MUST be restricted to explicitly specified contexts ([read this for more details](context.md)), which means we'd need a syntactic element to mark some _expression_ as an _observable expression_. For creating such construct, lets follow the example of `Promsie` flattening:

```js
const a = makeSomePromise(...)
const b = (await a) + 2
```
☝️ This wouldn't work, as we need to put this expression on the second line into an async context. The simplest way of doing that would be to use an asynchronous arrow function, i.e.
```js
const a = makeSomePromise(...)
const b = async () => (await a) + 2

/* Note that b is not eagerly evaluated (you need to call it) while a is. The correct code would basically be:
 * ```js
 * const a = makeSomePromise(...)
 * const b = (async () => (await a) + 2)()
 * ```
 * However, since observables are also lazily evaluated, we don't have to worry about 
 * this lazy-to-eager syntactic overhead for our problem.
 */
```
We could construct a similar construct for specifying _observable contexts_:
```
@ => <Some Expression>
```
The expression following `@ =>` would basically become an _observable context_, within it we could use a flattening operator, similar to `await`, for accessing values within observables:

```js
const a = makeSomeObservable(...)
const b = @ => @a + 2
```
