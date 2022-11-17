# Reactive Javascript

> This work started as [this gist](https://gist.github.com/loreanvictor/ef4fbc84e6be32adfb1404d6e0e7d05e) and I am now in the process of better organizing it within this repository.

Reactive programming in JavaScript is not easy, as it is not directly supported by the language itself. The community has tried to address this shortcoming via frameworks that try to solve it specifically for client side applications (like [React](https://reactjs.org)), and reactive programming utilities and libraries (the most famous being [RxJS](https://rxjs.dev)) that address this issue in isolation.

Nevertheless, the status quo still feels lacking for such a fundamental use case of JavaScript. Frameworks mostly need to bend the semantics of the language (e.g. [React components are NOT like other functions](https://reactjs.org/docs/hooks-overview.html#rules-of-hooks)) and libraries shift towards paradigms that increase code complexity (e.g. the FRP style of RxJS). Per 2021 State of JS Survey, amongst features missing from JavaScript, [native support for observables was ranked as fifth](https://2021.stateofjs.com/en-US/opinions/#currently_missing_from_js_wins).

<details open><summary>React Example</summary>

```jsx
import { useState } from 'react'

function Counter({ name }) {
  const [count, setCount] = useState(0) 
  // ☝️ this function can only be used in a React component.

  const color = count % 2 === 0 ? 'red' : 'blue'
  // ☝️ `color` is recalculated even when `count` has not changed.
  
  // 👇 a new function is defined every time `count` has a new value.
  //    a new function is also redefined whenever `name` changes.
  function incr() {
    setCount(count + 1)
  }
  
  return <div onClick={incr} style={{ color }}>
    { name || 'You' } have clicked {incr} times!
  </div>
}
```

</details>
<details><summary>RxJS Example</summary>

```js
import { map, fromEvent, scan } from 'rxjs'

// 👇 FRP style programming is not convenient for many developers.
//    JavaScript is NOT a functional language, so combination of these styles
//    inevitably increases code complexity.
const count = fromEvent(button$, 'click').pipe(scan(c => c + 1, 0))
const color = count.pipe(map(c => c % 2 === 0 ? 'red' : 'blue'))

count.subscribe(c => span$.textContent = c)
color.subscribe(c => div$.style.color = c)
```

</details>

<br>

This work is an invesitagtion of a potential syntactic solution to this shortcoming. The main idea is to be able to treat observable values as plain values in expressions, thus removing syntactic overhead required for conducting basic operations on observable values (re-evaluation of functional components in React, FRP-style programming with RxJS).

<details open><summary>React Example Reimagined</summary>

```jsx
function Counter({ name }) {
  let @count = 0
  // ☝️ this can be used anywhere, with the same meaning.
  
  const @color = (@count % 2 === 0) ? 'red' : 'blue'
  // ☝️ `color` is recalculated only when `count` has changed.

  // 👇 the click callback is defined once
  return(
    <div onclick={() => @count++} style={{ color }}>
      { name || 'You' } have clicked { count } times!
    </div>
  )
}
```

</details>

<details><summary>Same Example without JSX</summary>

```js
// No need for FRP style programming

let @count = 0
button$.addEventListener('click', () => @count++)

observe {
  span$.textContent = @count;
  if (@count % 2 === 0) {
    div$.style.color = 'red'
  } else {
    div$.style.color = 'blue'
  }
}
```

</details>

<details><summary>Same Example with DOM API Support</summary>

```js
let @count = 0
const @color = (@count % 2 === 0) ? 'red' : 'blue'

button$.addEventListener('click', () => @count++)
span$.textContent = count
div$.style.color = color
```

</details>

<br>

# Terminology

👉 For more precise definitions of various terms and expressions used in this repository regarding reactive programming and observables (such as _observable_, _shared observation_, _flattening_, etc.), [read the _definitions_ document](definitions.md). Generally speaking, semantics similar to that of [RxJS](https://rxjs.dev) are used, for example it is assumed that observables have the same interface as an RxJS [`Subscribable`](https://rxjs.dev/api/index/interface/Subscribable).

<br>

# The Idea

As mentioned above, the idea is syntactically [_flattening_ observables](flattening.md), i.e. adding new syntax that allows working with observables
directly within expressions by providing access to values they _wrap_. This is similar to how `Promise`s are flattened with the `await` keyword:

```js
const a = new Promise(...)
// ...

// Not flattened
a.then(_a => _a + 2)

// Flattened
(await a) + 2
```

<br>

We could use similar syntax, the `@` operator, for flattening observables:

```js
const a = makeObservable(...)
// ...

// Not flattened
a.pipe(map(_a => _a + 2))

// Flattened
@a + 2
```

<br>

Similar to how flattening `Promise`s can only occur in asynchronous contexts, flattening observables can also only occur in observable contexts ([read this for more on why](context.md)). This can be achieved with a construct similar to anonymous async functions, explicitly specifying boundaries of an observable context:

```js
const a = makeObservable(...)
const b = @ => @a + 2
```

<br>

> 👉 [Read this](flattening.md) for more details on the proposed syntax for observable flattening.

<br>

# Extensions

The proposed [observable flattening syntax](flattening.md) can be further extended with additional syntactic sugar, further simplifying common use cases where observables are used. Each of the following extensions can be considered and implemented independently, though they all depend on the original base syntax.

<br>

## Observable Creation

A common use case when handling observables is creating dependent observables, i.e. observables whose value is dependent on some source observables.
This can be simplified via an additional _observable creation syntax_:

```js
const a = makeObservable(...)
const b = makeAnotherObservable(...)

const @c = @a * 2 + @b
```
Which is shorthand for
```js
const c = @ => @a * 2 + @b
```
And would translate to:
```js
const c = combineLatest(a, b).pipe(map(([_a, _b]) => _a * 2 + _b))
```

<br>

> 👉 [Read this](creation.md) for more details on the proposed syntax for observable creation.

<br>

## Observation

Unlike `Promise`s, which are immediately executed, observables are _lazy_, which means they don't get executed until they are observed. To facilitate this, an additional syntactic construct can be created in form of a new keyword, `observe`:

```js
observe { console.log(@a) }
```

Which would be equivalent to:

```js
a.subscribe(_a => console.log(_a))
```

Or:

```js
(@ => { console.log(@a) }).subscribe()
```

This syntax can be further enhanced to allow handling errors that occur on execution path of an observation and finalize the whole process:

```js
observe {
  console.log(@a + 2)
} catch (error) {
  // an error has occured while computing new values for this observation,
  // which terminates the observation.
  console.log('Something went wrong ...')
} finally {
  // the observation is finished because all of its observable sources have
  // finished producing data.
  console.log('a is completed now.')
}
```
Which would be equivalent to:
```js
a.subscribe(
  _a => console.log(_a + 2),
  (error) => {
    console.log('Something went wrong ...')
  },
  () => {
    console.log('a is completed now.')
  }
)
```

<br>

> 👉 [Read this](observation.md) for more details on the proposed syntax for observation.

<br>

## Explicit Dependencies

In the [observable flattening syntax](flattening.md), we create observable expressions that are dependent on some other observables. In the proposed syntax these dependencies are implicitly detected:
```js
const c = @ => @a * 2 + @b
```

It can be particularly useful to be able to explicitly specify these depencies as well. For example, you might want to have run some side effect without using the observed value:
```js
// implicit dependencies
const log = @ => { @click; console.log('CLICKED!') }
```
```js
// explicit dependencies
const log = @(click) => console.log('CLICKED!')
```

<br>

It can also increase readability of the code in case of complex expressions:
```js
// implicit dependencies
const dependent = @ => {
  // some code here
  somethingDependsOn(@a)
  
  // some more code here
  somethingElseDependsOn(@b)
  
  // ...
}
```
```js
// explicit dependencies
const dependent = @(a, b) => {
  // some code here
  somethingDependsOn(@a)
  
  // some more code here
  somethingElseDependsOn(@b)
  
  // ...
}
```

<br>

And it can be used as a method of _passively tracking_ some other observables, i.e. using their latest value without re-calculating and re-emitting the expression when they emit new values:

```js
const c = @(a) => @a * 2 + @b
// ☝️ c is only re-evaluated when a has a new value, though latest value of b will be used.
```

<br>

> 👉 [Read this](explicit-dependencies.md) for more details on the proposed syntax for explicit dependencies.

<br>

## Cold Start

In many cases it is helpful to assume a _default_ value for an observable before it emits its first value (for each observation). With the [proposed flattening operator `@`](flattening.md), the observable expression won't be calculated until each observable emits at least once. This can be resolved by adding a cold start operator `@?`, that causes the observable to emit `undefined` initially upon observation, allowing addition of default values:

```js
const greeting = new Subject()
const name = new Subject()

const msg = @ => (@?greeting ?? 'Hellow') + ' ' + (@?name ?? 'World')
// ☝️ msg will be 'Hellow World' initially.

greeting.next('Hallo')
// ☝️ msg will be 'Halo World'.

name.next('Welt')
// ☝️ msg will be 'Halo Welt'.
```

Which would be equivalent to:

```js
const msg = combineLatest(
  greeting.pipe(startWith(undefined)),
  name.pipe(startWith(undefined)),
).pipe(map(([_greeting, _name]) => (_greeting ?? 'Hellow') + ' ' + (_name ?? 'World')))
```

<br>

> 👉 [Read this](cold-start.md) for more details on the proposed syntax for cold start.

<br>
