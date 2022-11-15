# Observable Flattening

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

<br>

We could use a similar construct for specifying _observable contexts_:
```
@ => <Some Expression>
```

> ❓ _Why this syntax? Why not a new function modifier like `async`?_
> 
> Observable expressions, though lazily evaluated, are NOT functions. An async function is NOT a `Promise` itself,
> it rather yields a `Promise` when you call it. Observable expressions, on the other hand, ARE observables themselves, not functions
> of observables.


<br>

The expression following `@ =>` would become an _observable context_, within it we could use a flattening operator, similar to `await`, for accessing values within observables:

```js
const a = makeSomeObservable(...)
const b = @ => @a + 2
```

> ❓ _Why this syntax? Why not a keyword like `await`?_
> 
> A keyword such as `await` would mean the _flattening operator_ could also operate on expressions. For async expressions, such
> inner-expressions won't cause semantic ambiguity, since the whole expression, including the inner expression, is evaluated ONCE.
> Observable expressions, however, are evaluated potentially many times, which would result in the inner expression also being evaluated
> each time, resulting in a new observable the expression is dependent on, which is NEVER the intended behavior.

<br>

### No Out-of-context Flattening
The flattening operator `@` MUST be limited to observable contexts, and usage outside of such contexts needs to be a syntax error:
```js
const a = makeSomeObservable(...)
const b = @a + 2 // ❌ SYNTAX ERROR!
```
[Read this](context.md) for more details on why. In short, out-of-context usage creates semantic ambiguity about the boundaries of the observable expression, e.g. for the following code:
```js
console.log(@a + 2)
```
It cannot be determined wheter an observable (`@a + 2`) should be logged ONCE, or whether new values of `a` (plus 2) should be logged each time `a` has a new value. Furthermore, without explicit disambiguation, leaning either way would either [result in a semantic contradiction or violate some essential syntactic invariance](context.md).

<br>

### Only Flatten Out-of-context Identifiers

The flattening MUST BE limited to _identifiers_ defined outside the current observable context:
```js
// ❌ SYNTAX ERROR!
const b = @ => @makeSomeObservable(...) + 2
```
This is because observable expressions yield _dependent observables_, i.e. observables that emit new values whenever the source observables they depend on emit values. So for example in the following case:
```js
const a = makeSomeObservable(...)
const b = makeSomeObservable(...)
const c = @ => @a + @b
```
`c` has a new value every time `a` or `b` have new values, and this new value is calculated by re-evaluating the observable expression that is defining `c`. However, in our previous, erroneous case, the observable `b` is dependent on changes _every time_ the source observable has a new value, as the dependency is described using an expression within the observable expression itself. The limit to identifiers defined outside of the current context ensures that dependencies are stable and don't change with each re-evaluation of the observable expression.

An exception to this rule would be chain-flattening:
```js
// What we want to do:
// start a new timer whenever some button is clicked,
// and display the value of the last timer.

const click = observableFromClick(...)
const timers = @ => makeAnObservableTimer(...)
const message = @ => `Latest timer is: ${@@timers}`
```
Here, `timers` is an observable whose values are observables themselves, so `@timers` is still an observable. `@@timers` can unambiguously be resolved to the latest value of the latest observable emitted by `timers`, which means `message` is still only dependent on `timers` and its dependencies do not get changed with every re-evaluation.

<br>

## Transpilation

The proposed solution is merely syntactical: in other words it DOES NOT enable new things to do, it only enables rewriting existing code in a simpler, more readable manner. Generally, for any expression `E` with _n_ free variables, and `a_1`, `a_2`, ..., `a_n` being identifiers not appearing in `E`, the following schematic code of the new syntax:

```js
@ => E(@a_1, @a_2, ..., @a_n)
```

Could be transpiled to:

```js
combineLatest(a_1, a_2, ..., a_n)
  .pipe(
    map(([_a_1, _a_2, ..., _a_n]) => E(_a_1, _a_2, ..., _a_n))
  )
```
In other words, the expression yields a new observable that whenever either one of `a_1`, `a_2`, ..., `a_n` have a new value, then `E` is re-evaluated with all of the latest values and the resulting observable assumes (emits) the resulting value. `E` can also be a _function body_, i.e.
```
{
  <STATEMENT>
  <STATEMENT>
  ...
  
  [return <EXPRESSION>]
}
```
where `@a_1`, ..., `@a_n` appear in the statements and the optional return expression, with the same transpilation.

> For semantic clarity, in these examples, behavior of [`combineLatest()`](https://rxjs.dev/api/index/function/combineLatest) 
> and [`map()`](https://rxjs.dev/api/index/function/map) from [RxJS](https://rxjs.dev) is assumed, though for an actual implementation,
> pehraps more efficient and light-weight libraries can be utilized. There is also an implicit assumption here that is not mentioned in the examples
> for sake of maintaing code clarity: if any `a_i` is not an observable, then `of(a_i)` is used for the transpilation. This can be conducted with a simple
> normalizing operator where all arguments are passed through it:
> ```js
> const normalize = o => (o instance of Observable) ? o : of(o)
> ```

<details><summary>Example</summary>
  
```js
// Proposed syntax:
const a = interval(100)
const b = @ => Math.floor(@a / 10)
```
```js
// Transpilation:
const a = interval(100)
const b = combineLatest(a).pipe(map(([_a]) => Math.floor(_a / 10)))
```

</details>
<details><summary>Another Example</summary>

```js
// Proposed syntax:
const a = interval(100)
const b = @ => {
  const seconds = Math.floor(@a / 10)
  const centi = @a - seconds
  
  return `${seconds}:${centi}`
}
```
```js
// Transpilation:
const a = interval(100)
const b = combineLatest(a).pipe(
  map(([_a]) => {
    const seconds = Math.floor(_a / 10)
    const centi = _a - seconds
  
    return `${seconds}:${centi}`
  })
)
```

</details>
<details><summary>Another Example</summary>

```js
// Proposed syntax
const a = interval(100)
const b = interval(200)
const c = @ => @a + @b
```
```js
// Transpilation:
const a = interval(100)
const b = interval(200)
const c = combineLatest(a, b).pipe(map(([_a, _b]) => _a + _b))
```

</details>
<details><summary>Another Example</summary>

```js
// Proposed syntax
const a = interval(100)
const b = interval(200)
const c = @ => {
  console.log('New Value!')
  
  return @a + @b
}
```
```js
// Transpilation:
const a = interval(100)
const b = interval(200)
const c = combineLatest(a, b).pipe(
  map(([_a, _b]) => {
    console.log('New Value!')
    
    return _a + _b
  })
)
```

</details>

For transpilation of chain flattening, we can flatten the observable before combining it with other observables, i.e. we could transpile the  general form:
```js
@ => E(@@a_1, @a_2, ..., @a_n)
```
to:
```js
combineLatest(a_1.pipe(switchAll()), a_2, ..., a_n)
  .pipe(
    map(([__a_1, _a_2, ..., _a_n]) => E(__a_1, _a_2, ..., _a_n))
  )
```
With the same transpilation for any arbitrary `a_i`. Additionally, for larger flattening chains, we just need to further flatten the observable, i.e. turn the following:
```js
@@@@a_5
```
to:
```js
a_5.pipe(switchAll(), switchAll(), switchAll())
```

<details><summary>Example</summary>

```js
// Proposed syntax:
const click = fromEvent($btn, 'click')
const timers = @ => { @click; return interval(1000) }
const message = @ => `Latest timer is: ${@@timers}`
```
```js
// Transpilation:
const click = fromEvent($btn, 'click')
const timers = combineLatest(click).pipe(map(([_click]) => { _click; return interval(1000) })
const message = combineLatest(timers.pipe(switchAll())).pipe(map(([__timers]) => `Latest timer is ${__timers}`))
```

</details>
