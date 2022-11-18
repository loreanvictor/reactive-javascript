# Observable Creation

Arguably one of the most common use cases for [observable expressions](/flattening.md) is creating _computed_ or _dependent_ observables:

```js
const c = @ => @a + @b
```

The [porposed syntax for observable flattening](/flattening.md) can be expanded to further streamline and flatten such declerations:

```js
const @c = @a + @b
```

Generally, for any expression `E`, we can have the short-hand:

```js
const @c = E
```

Be desugared to:

```js
const c = @ => E
```

Note that `E` in the above examples _IS_ an [observable context](/context.md), which allows usage of the flattening operator `@`.

<br>

## Higher-Level Observables

The proposed syntax can also be extended to streamline creation of higher-level observables. Since chain-flattening can only apply to identifiers, only taking the [observable flattening syntax](/flattening.md) into account, you would need to create intermediate variables referencing the higher-level observables in order to be able to use the flattened version:

```js
// 👇 create a new timer for each click of a button
const click = fromEvent($btn, 'click')
const @timers = interval((@click, 1000))
const @timer = @@timers
```

This can be simplified by allowing chain flattening during decleration:

```js
const click = fromEvent($btn, 'click')
const @@timer = interval((@click, 1000))
```

Generally for any expression `E`, we could desugar this code:

```js
const @@x = E
```

to the following:

```js
const @X = E
const @x = @@X
```

With similar desugaring for longer chains:

```js
// Proposed syntax
const @@@@x = E
```
```js
// Desugarred code
const @X = E
const @x = @@@X
```

<br><br>
