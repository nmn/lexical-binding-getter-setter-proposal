# lexical-binding-getter-setter-proposal

A proposal for extending JS syntax to support lexically scoped getters and
setters for variable bindings.

## Motivation

In Javascript, we can have getter and setter functions for object properties so
that we can run arbitrary code when that property is read or written to:

```js
const obj = {
    value: 1,
    get doubleVal() {
        return this.value * 2
    },
    set doubleVal(v) {
        this.value = v / 2
    },
}

console.log(obj.doubleVal) // 2
obj.doubleVal = 8

console.log(obj.doubleVal) // 8
console.log(obj.value) // 4
```

However, something like this is not possible for normal variable bindings:

```js
function foo () {
    let value = 1;

    let double = {
        get() {
            return value * 2;
        },
        set(v) {
            value = v / 2;
        }
    }
}
```

This lack is made obvious in many different UI libraries but is most obvious
in cases where signals are used, such as Solid.js, and Svelte.js

## Inpiration

### Svelte 5 Runes

The closest inpiration for this proposal is Svelte 5's runes, which is enables
a similar syntax by leveraging a compiler. 

```js
let state = $state(0)

state = 1 // desugars to state.set(1)
console.log(state) // desugars to console.log(state.get())
```

This makes the common use-case of reading or writing to the signal concise.
However, due to the lack of proper syntax support in JS, certain use-cases,
such as passing the entire signal as an argument are a bit more verbose:

```js
let state = $state(0)

takesTheSignal({
    get() {return state},
    set(v) {state = v},
});
```

### Swift Getters and Setters

Swift has a similar features which lets you define getters and setters for
simple variables:

```swift
var value: Float = 0
var doubleValue: Float {
    get { value * 2 }
    set { value = newValue / 2 }
}

print(doubleValue) // 2
doubleValue = 8

print(doubleValue) // 8
print(value) // 4
```

## Proposal

We propose adding a new keyword that can be used instead of `let` or `var`
to define special variable bindings that would indicate that the value is
a special object with getters and setters instead.

This new keyword should be `bind` and can be used similarly to the other
similar proposal for adding the `using` keyword to the language.

With this new proposal, the code above can be simplified to:

```js
bind doubleValue = {
    innerValue: 1,
    get() {
        return this.innerValue * 2
    },
    set(v) {
        this.innerValue = v / 2
    },
}

console.log(doubleValue) // 2
doubleValue = 8

console.log(doubleValue) // 8
console.log(#doubleValue.innerValue) // 4
```

### Details

#### Read-only, Write-only or both

Similar to the proposal for `using`, using the `bind` keyword to create a
variable binding would require the value to have the `get` or `set` properties
or both. Skipping `set`, would make the binding read-only and trying to set
the value would result in a runtime exception.

#### Scoped Lexically

Reading or writing to this binding will result in the `get` or `set` function
to be called respectively. By design, this will only work in the same lexical
scope as where the binding is defined. Note how calling `console.log(doubleValue)`
logs the result of calling `get()` and not the object itself.

#### Accessing the underlying object

Since the `doubleValue` binding is proxy to the `get()` and `set()` methods of
the associated object, in order to access the object itself, `#doubleValue` can
be used.

The `#` character is chosen to mirror it's usage to represent private properties
in a class.

Using, `#doubleValue`, you get a reference to the object itself to be able to
read other properties manually. I.e.:

```js
console.log(#doubleValue.innerValue)
```

This same capability can be used to pass the entire binding around:

```js
functionThatTakesBinding(#doubleValue)
```

#### Usage for function arguments

Since functions can accept objects as arguments and arguments are a special
kind of variable bindings, it should be possible to use the `bind` keyword
with arguments as well:

```js
function functionThatTakesBinding(bind binding) {
    console.log(binding); // calling `.get()` behind the scenes
    binding = 20; // calling `.set(20)` behind the scenes
}
```

## Examples

### React

Before:

```js
const [count, setCount] = useState(0)

return <div>
  {count}
  <button onClick={() => {setCount(c => c + 1)}}>Incr</button>
  <OtherCounter count={count} setCount={setCount} />
</div>
```

After:

```js
bind count = useState(0)

return <div>
  {count}
  <button onClick={() => { count++ }}>Incr</button>
  <OtherCounter countBinding={#count} />
</div>
```

### Solid

Before:

```js
const [count, setCount] = createSignal(0)

return <div>
  {count()}
  <button onClick={() => {setCount(c => c + 1)}}>Incr</button>
  <OtherCounter count={count()} setCount={setCount} />
</div>
```

After:

```js
bind count = createSignal(0)

return <div>
  {count}
  <button onClick={() => { count++ }}>Incr</button>
  <OtherCounter countBinding={#count} />
</div>
```

### Svelte 5

Before:

```js
export function createCounter() {
	let count = $state(0);

    // other code

	return {
		get count() { return count },
		set count(newValue) { count = newValue },
   };
}
```

After:

```js
export function createCounter() {
	bind count = signal(0);

    // other code

	return #count
}
```