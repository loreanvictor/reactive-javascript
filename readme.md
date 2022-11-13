# Reactive Javascript

> This work started as [this gist](https://gist.github.com/loreanvictor/ef4fbc84e6be32adfb1404d6e0e7d05e) and I am now in the process of better organizing it within this repository.

Reactive programming in JavaScript is not easy, as it is not directly supported by the language itself. The community has tried to address this shortcoming via frameworks that try to solve it specifically for client side applications (like [React](https://reactjs.org), and reactive programming utilities and libraries (the most famous being [RxJS](https://rxjs.dev)) that address this issue in isolation.

In any case, the status quo still seems unnecessarily convoluted for such a fundamental use case of JavaScript. Frameworks mostly need to bend the semantics of the language (e.g. [React components are NOT like other functions](https://reactjs.org/docs/hooks-overview.html#rules-of-hooks)) or shift towards paradigms that increase code complexity (e.g. the FRP style of RxJS).


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

This work is an invesitagtion of what potential syntactic solutions to this shortcoming would look like. The main idea is to be able to treat observable values as plain values in expressions, thus removing syntactic overhead required for conducting basic operations on observable values (re-evaluation of functional components in React, FRP-style programming with RxJS).

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

## Definitions

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
A mechanism for accessing values of observables. Imagine the higher-order observable from previous example (a stream of timers), you might decide to display the value of the last timer, or you might want to merge incoming values of all timers and display that. Either way, you are _flatenning_ a higher-order observable (a stream of timers) into a lower-order observable (a stream of numbers).

> The key idea being investigated in this work is that of _flatenning_ observables using additional syntax, which allows for treating observables like plain values.

