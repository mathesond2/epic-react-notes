React.useEffect is a built-in hook that allows you to run some custom code after React renders (and re-renders) your component to the DOM. It accepts a callback function which React will call after the DOM has been updated

## lazy state initialization
ex:
```js
function Greeting({initialName = ''}) {
  const [name, setName] = React.useState(window.localStorage.getItem('name') || initialName)

  React.useEffect(() => {
    window.localStorage.setItem('name', name)
  })

  function handleChange(event) {
    setName(event.target.value)
  }

  return (
    <div>
      <form>
        <label htmlFor="name">Name: </label>
        <input value={name} onChange={handleChange} id="name" />
      </form>
      {name ? <strong>Hello {name}</strong> : 'Please type your name'}
    </div>
  )
}
```
^ this is normal. Every time that `handleChange` event happens, we re-render the component via the state update `setName`. This is fine,but imagine if we needed to do a very expensive initial render (ex: local storage itself can be slow):

`useState` allows you to pass a function instead of the actual value, and then **it will only call that function to get the state value when the component is rendered the first time**.

 So you can go from this:
 ```js
 React.useState(someExpensiveComputation())
 ```
 To this:
 ```js
React.useState(() => someExpensiveComputation())
 ```
And the `someExpensiveComputation` function will only be called when it’s needed!

**TL;DR: for lazy state initiatization (for expensive operations), pass a fn rather than a value to `useState()`**

NOTE:
this:
```js
React.useState(() => someExpensiveComputation())
```
is the same as this:
```js
React.useState(someExpensiveComputation);
```


## custom hooks
we may want to encapsulate some logic and keep it elsewhere for reuse upon diff places. we can create custom hooks for this.

hooks are just functions. components are just functions.

since components can have their own state and useEffect hooks, functions can have their own state and their own useEffect hooks too.
```js
function useLocalStorageState(key, defaultValue = '') {
  const [state, setState] = React.useState(
    () => window.localStorage.getItem(key) || defaultValue,
  )

  React.useEffect(() => {
    window.localStorage.setItem(key, state)
  }, [key, state])

  return [state, setState]
}

  const [name, setName] = useLocalStorageState('name', initialName)

    function handleChange(event) {
    setName(event.target.value)
  }

  return (
    <div>
      <form>
        <label htmlFor="name">Name: </label>
        <input value={name} onChange={handleChange} id="name" />
      </form>
      {name ? <strong>Hello {name}</strong> : 'Please type your name'}
    </div>
  )
```

so a custom hook allows you to encapsulate some functionality for reuse and provides a familiar api to interact with it.


interesting...when creating a custom hook, kent first implements the function as it'd be called ex: `const [name, setName]syncLocalStorageWithState();`...this is great because it provides an initial understanding for how the fn will be used and whaat it needs


note: a custom hook is just a function with hooks inside it.

- Managed State: State that you need to explicitly manage
- Derived State: State that you can calculate based on other state
```js
//managed state
const [squares, setSquares] = React.useState(Array(9).fill(null));

//derived state
const nextValue = calculateNextValue(squares);
const winner = calculateWinner(squares);
```
^ what makes this useful is that we could create state values for `nextValue` and `winner` but then we will have to make sure that we are always updating `nextValue`,`winner`, and `squares` together so they stay in sync. Instead of trying to sync various states, just derive the state from the necessary 'true state' (aka in our instance, `squares`). since these consts run on every re-render, we'll never have to worry about syncing our state and our component becomes much simpler.

**TL;DR dont sync state, derive state constants from existing state. This makes your state mgmt simpler.**

## http calls within useEffect
One important thing to note about the useEffect hook is that you cannot return anything other than the cleanup function. This has interesting implications with regard to async/await syntax:
```js
// this does not work, don't do this:
React.useEffect(async () => {
  const result = await doSomeAsyncThing()
  // do something with the result
})
```
The reason this doesn’t work is because when you make a function async, it automatically returns a promise (whether you’re not returning anything at all, or explicitly returning a function). This is due to the semantics of async/await syntax. So if you want to use async/await, the best way to do that is like so:
```js
React.useEffect(() => {
  async function effect() {
    const result = await doSomeAsyncThing()
    // do something with the result
  }
  effect()
})
```
This ensures that you don’t return anything but a cleanup function.


## putting state as an object
You’ll notice that we’re calling a bunch of state updaters in a row. This is normally not a problem, but each call to our state updater can result in a re-render of our component. React normally batches these calls so you only get a single re-render, but it’s unable to do this in an asynchronous callback (like our promise success and error handlers).

So you might notice that if you do this:
```js
setStatus('resolved')
setPokemon(pokemon)
```

You’ll get an error indicating that you cannot read image of null. This is because the setStatus call results in a re-render that happens before the setPokemon happens.

Error