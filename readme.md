# Reactive Javascript

> This work started as [this gist](https://gist.github.com/loreanvictor/ef4fbc84e6be32adfb1404d6e0e7d05e) and I am now in the process of better organizing it within this repository.

Reactive programming in JavaScript is not easy, as it is not directly supported by the language itself. The community have tried to address this shortcoming via frameworks that try to solve it specificall for client side applications (like [React](https://reactjs.org), and reactive programming utilities and libraries (the most famous being [RxJS](https://rxjs.dev)) that try to fill in the gap in isolation.

In any case, the status quo still seems unnecessarily convoluted for such a fundamental use case of JavaScript. Frameworks mostly need to bend the semantics of the language (for example, in React, components are not like other functions, we have hooks, etc) or shift towards paradigms that increase code complexity (e.g. the functional reactive programming style of RxJS, which won't even be better supported via proposals such as [the pipeline proposal](https://github.com/tc39/proposal-pipeline-operator)).


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
