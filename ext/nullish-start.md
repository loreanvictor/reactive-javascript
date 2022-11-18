# Nullish Start

An [observable expression](/flattening.md) won't emit any values until all of the observables it depends on have emitted at least once (for each observation). However, in many cases it is desirable to have _default values_ for each source observable:

```js
const input = fromEvent($input, 'input')
const msg = @ => `Hellow ${@input.value}!`

observe { $div.textContent = @msg }
// ☝️ the element won't display any values until there is an input event,
//    while a better user experience would be displaying some default text.
```

Without further syntactic support, the default value could be achieved like this:

```js
const input = fromEvent($input, 'input').pipe(startWith({ value: 'Anon' }))
const msg = @ => `Hellow ${@input.value}!`

observe { $div.textContent = @msg }
// ☝️ now we start with 'Hellow Anon!'
```

If the flattened observable value (`@input.value`) would emit upon observation with a nullish value, we could use JavaScript's own syntactic support for default values (such as the [nullish coalescing operator](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Nullish_coalescing)) for handling our use case in a more readable way. This can be achieved by using a _nullish flattening_ operator `@?` instead of the default flattening operator `@`:

```js
const input = fromEvent($input, 'input')
const msg = @ => `Hellow ${@?input?.value ?? 'Anon'}!`
```

<br>

## Transpilation

For an expression `E` with _n_ free variables, and `a_1`, `a_2`, ..., `a_n` being identifiers not appearing in `E`, the following general form:

```js
@ => E(@?a_1, @a_2, ..., @a_n)
```
Could _roughly_ be transpiled to:
```js
combineLatest(a_1.pipe(startWith(undefined)), a_2, ..., a_n).pipe(
  map(([_a_1, _a_2, ..., _a_n]) => E(_a_1, _a_2, ..., _a_n))
)
```
With the same transpilation for any arbitrary `a_i`.

<br>

> 💡 _On Transpilation_
>
> For semantic clarity, [RxJS](https://rxjs.dev)'s operators are assumed, though for implementation, more light-weight libraries and protocols can
> be utilized. Read [this](/flattening.md#transpilation) for more info.

<br>

The outlined transpilation will result in `undefined` always being emitted initially, even when the observable emits values synchronously upon observation (e.g. a [`ReplaySubject`](https://rxjs.dev/api/index/class/ReplaySubject)). This can be fixed so that the _nullish start_ only happens when the observable does not immediately emit a value upon observation, for example by using a new `nullishStart()` operator instead of RxJS's own [`startWith()`](https://rxjs.dev/api/index/function/startWith) operator:

```ts
// TypeScript syntax for further clarity

export const nullishStart = () => <T>(source: Observable<T>) => {
  return new Observable<T | undefined>(observer => {
    let emitted = false
    const subscription = source.subscribe(
      value => { emitted = true; observer.next(value) },
      error => observer.error(error),
      () => observer.complete(),
    )
    
    if (!emitted) {
      observer.next(undefined)
    }
    
    return subscription
  })
}
```
Which could be then used for a more precise transpilation:
```js
combineLatest(a_1.pipe(nullishStart()), a_2, ..., a_n).pipe(
  map(([_a_1, _a_2, ..., _a_n]) => E(_a_1, _a_2, ..., _a_n))
)
```

<details><summary>Example</summary>

```js
// Proposed syntax:
const input = fromEvent($input, 'input')
const msg = @ => `Hellow ${@?input?.value ?? 'Anon'}!`

observe { $div.textContent = @msg }
```
```js
// Transpilation:
const input = fromEvent($input, 'input')
const msg = combineLatest(input.pipe(nullishStart())).pipe(
  map(([_input]) => `Hellow ${_input?.value ?? 'Anon'}`)
)

msg.subscribe(
  ([_msg]) => $div.textContent = _msg
)
```

</details>

<br>

This can also be combined with chained flattening, by applying the _nullish start_ after all the flattening has occured (so the resultant observable emits a nullish value even if the higher-order observables have not yet emitted). This means the following general form:

```js
@ => E(@@@?a_1, @a_2, ..., @a_n)
```
Can be transpiled to:
```js
combineLatest(
  a_1.pipe(switchAll(), switchAll(), nullishStart()),
  a_2,
  ...,
  a_n
).pipe(map(([___a_1, _a_2, ..., _a_n]) => E(___a_1, _a_2, ..., _a_n)))
```

<br><br>
