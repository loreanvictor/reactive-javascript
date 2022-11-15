# Reactive Javascript

> This work started as [this gist](https://gist.github.com/loreanvictor/ef4fbc84e6be32adfb1404d6e0e7d05e) and I am now in the process of better organizing it within this repository.

Reactive programming in JavaScript is not easy, as it is not directly supported by the language itself. The community has tried to address this shortcoming via frameworks that try to solve it specifically for client side applications (like [React](https://reactjs.org), and reactive programming utilities and libraries (the most famous being [RxJS](https://rxjs.dev)) that address this issue in isolation.

Nevertheless, the status quo still feels lacking for such a fundamental use case of JavaScript. Frameworks mostly need to bend the semantics of the language (e.g. [React components are NOT like other functions](https://reactjs.org/docs/hooks-overview.html#rules-of-hooks)) and libraries shift towards paradigms that increase code complexity (e.g. the FRP style of RxJS). Per 2021 State of JS Survey, amongst features missing from JavaScript, [native support for observables was ranked as fifth](https://2021.stateofjs.com/en-US/opinions/#currently_missing_from_js_wins).


```jsx
import { useState } from 'react'

function Counter() {
  const [count, setCount] = useState(0)
  const color = count % 2 === 0 ? 'red' : 'blue'
  
  function incr() {
    setCount(count + 1)
  }
  
  return <div onClick={incr} style={{ color }}>
    You have clicked {incr} times!
  </div>
}
```
```js
import { map, fromEvent, scan } from 'rxjs'

const count = fromEvent(button$, 'click').pipe(scan(c => c + 1, 0))
const color = count.pipe(map(c => c % 2 === 0 ? 'red' : 'blue'))

count.subscribe(c => span$.textContent = c)
color.subscribe(c => div$.style.color = c)
```

<br>

This work is an invesitagtion of a potential syntactic solution to this shortcoming. The main idea is to be able to treat observable values as plain values in expressions, thus removing syntactic overhead required for conducting basic operations on observable values (re-evaluation of functional components in React, FRP-style programming with RxJS).

```jsx
// No need for re-running functional components,
// or even using functional components

let @count = 0
const @color = (@count % 2 === 0) ? 'red' : 'blue'

render(
  <div onclick={() => @count++} style={{ color }}>
    You have clicked { count } times!
  </div>
)
```
```js
// without JSX
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
```js
// Or potentially, with also support from DOM APIs:

let @count = 0
const @color = (@count % 2 === 0) ? 'red' : 'blue'

button$.addEventListener('click', () => @count++)
span$.textContent = count
div$.style.color = color
```

<br>

# Definitions

### 👉 Observable
A value that changes in response to external events (passage of time, user input, Web Sockets, etc), or a side effect (change in some other program state, e.g. a console log) that is executed in response to such external events, or combinations of both. [Read this](https://rxjs.dev/guide/observable) for a more in depth intro.

> A timer is an observable, so is a state in a React component.

### 👉 Observation
An observation is an execution of an observable (basically a [`Subscription`](https://rxjs.dev/guide/subscription) in RxJS lingo). An observable won't do anything unless it is observed, which results in an observation which can be terminated at any point.

### 👉 Shared Observables
An observable might produce independent values and execute independent side effects for different observations. However, if it produces the same values and executes the same side effects for all of its observations, then it is _shared_.

> For shared observables, side effects are executed for each value / event. For non-shared observables, side effects are executed for each value / event for each observation.

### 👉 Higher-Order Observables
You can have observables of observables: Imagine that you have a button, and you start a new timer for each click of this button. The click event can be represented as an observable that is being mapped to a stream of new timers, each observable themselves.

### 👉 Flattening
A mechanism for accessing values wrapped within some other container such as observables. Imagine the higher-order observable from previous example (a stream of timers), you might decide to display the value of the last timer, or you might want to merge incoming values of all timers and display that. Either way, you are _flatenning_ a higher-order observable (a stream of timers) into a lower-order observable (a stream of numbers).

> Flatenning is not unique to observables. For example `Promise`s are always automatically flattened: you can have a _plain value_, or you can have a _promise of a plain value_, or a _promise of a promise of a plain value_, so on. If you use `.then()` method, then a _promise of a promise_ is automatically flattened:
> 
> ```js
> (new Promise(r => r(2)).then(console.log)
> // > 2
>
> (new Promise(
>   r => r(new Promise(
>     R => R(2)
>   ))
> )).then(console.log)
> // > 2
> // ☝️ It does not log a `Promise` object, it logs the value of the promise within.
> ```

<br>

# Core Idea
The main idea is to add a syntax to _flatten_ observables so that their values is usable in expressions just as plain values are. This is similar to what `await` syntax does for `Promise`s: without it, you need to pass handler functions to be able to access values wrapped within a `Promise`:

```js
const a = new Promise(...)
const b = a.then(_a => _a + 2)
```

With the `await` keyword, `Promise`s get flattened, so their value can be used in normal expressions:

```js
const a = new Promise(...)
const b = (await a) + 2
```

<br>

Similarly, we could have a syntactic solution for flattening observables:

```js
const a = makeObservable(...)
const b = a.pipe(map(_a => _a + 2))
```
```js
const a = makeObservable(...)
const b = @a + 2
```

<br>

Similar to how flattening `Promise`s can only occur in asynchronous contexts, flattening observables can also only occur in observable contexts ([read this for more on why](context.md)). This can be achieved with a construct similar to anonymous async functions, explicitly specifying boundaries of an observable context:

```js
const a = makeObservable(...)
const b = @ => @a + 2
```

Which _can_ be further simplified by introducing an _observable creation_ construct:

```js
const a = makeObservable(...)
const @b = @a + 2

// ☝️ this is shorthand for
// const b = @ => @a + 2
```

Similar to an anonymous function, the observable _"function"_ can also have a complete function body:

```js
const b = @ => { return @a + 2 }
```

Which allows executing side effects as well:

```js
const b = @ => {
  console.log('new value: ' + @a)
  return @a + 2
}
```

> 👉 [Read this](flattening.md) for more details on the proposed syntax.

<br>

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
