# Definitions

Here are some definitions of some of the terms used in this repository.

<br>

## 👉 Observable
A value that changes in response to external events (passage of time, user input, Web Sockets, etc), or a side effect (change in some other program state, e.g. a console log) that is executed in response to such external events, or combinations of both. [Read this](https://rxjs.dev/guide/observable) for a more in depth intro.

> A timer is an observable, so is a state in a React component.

<br>

## 👉 Observation
An observation is an execution of an observable (basically a [`Subscription`](https://rxjs.dev/guide/subscription) in RxJS lingo). An observable won't do anything unless it is observed, which results in an observation which can be terminated at any point.

<br>

## 👉 Shared Observables
An observable might produce independent values and execute independent side effects for different observations. However, if it produces the same values and executes the same side effects for all of its observations, then it is _shared_.

> For shared observables, side effects are executed for each value / event. For non-shared observables, side effects are executed for each value / event for each observation.

<br>

## 👉 Higher-Order Observables
You can have observables of observables: Imagine that you have a button, and you start a new timer for each click of this button. The click event can be represented as an observable that is being mapped to a stream of new timers, each observable themselves.

<br>

## 👉 Flattening
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
<br>
