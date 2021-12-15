# useReducer
- separate state logic from the components that make the state changes
- if you have multiple elements of state that typically change together, bundle them as in object
^ where useReducer comes in handy

```js
//here we have the current state and the 'action' (whatever the dispatch fn is called with...in this case 'event.target.value')
function nameReducer(previousName, newName) {
  return newName
}

const initialNameValue = 'Joe'

function NameInput() {
  /*
  so this is essentially the same as useState but
  */
  const [name, setName] = React.useReducer(nameReducer, initialNameValue)
  const handleChange = event => setName(event.target.value)
  return (
    <>
      <label>
        Name: <input defaultValue={name} onChange={handleChange} />
      </label>
      <div>You typed: {name}</div>
    </>
  )
}
```

functions can have useEffects within them? I figured they were top level. I guess this makes sense considering custom hooks are just regular functions.

something I didnt know...ternaries are possible within spread operator:
```js
const eyes = (eyeMovement) => ({
  color: "blue"
  ...(typeof eyeMovement === 'function' ? eyeMovement() : eyeMovement)
})
```
^ wild

look at this:
```js
function countReducer(state, action) {
  const {type, step} = action
  switch (type) {
    case 'increment': {
      return {
        ...state,
        count: state.count + step,
      }
    }
    default: {
      throw new Error(`Unsupported action type: ${action.type}`)
    }
  }
}

function Counter({initialCount = 0, step = 1}) {
  const [state, dispatch] = React.useReducer(countReducer, {count: initialCount})

  const {count} = state;
  const increment = () => dispatch({type: 'increment', step});
  return <button onClick={increment}>{count}</button>
}
```
so basically, dont worry about the `dispatch()` fn, worry about its args. thats where the complexity lies and realistically, we just feed the args to the reducer and that updates state based on the action type. voila.


# useCallback
everytime a component has to re-render, it recreates the functions within it as a brand new function. if that function is a prop to a child component, it re-renders that child comp as well, even if the function itself hasnt changed.

ex: imagine you have a function that relies on one piece of local state. another piece of local state changes, your function re-renders. This can be expensive. if that function is a prop to a child comp, it becomes more expensive...at this point we're re-rendering a function because of a piece of local state that it doesnt even know about. we can prevent these unnecessary re-renders.

useCallback hook makes it so its not going re-render the callback unless certain things change (ex: the state it acutally listens to/needs):

```js
const [number, setNumber] = useState(1);
const [dark, setDark] = useState(false);

const getItems = () => {
  return [number, number + 1, number + 2]
}

return (
  <h1>hi</h1>
  <List getItems={getItems}>
)
```

List.tsx
```js
export default List({getItems}) {
  useEffect(() => {
    //do something here

  },[getItems]);

  return (
    //some jsx
  )
}
```
^ so whenever we update `dark` state, we're unnecessarily re-creating  `getItems` and also re-rendering the `List` component and correspondingly running the useEffect inside of it.

useCallback allows us to memoize the fn:
```js
const getItems = useCallback(() => {
  return [number, number + 1, number + 2]
}, [numbers])
```
^ so now everytime <App> updates, `getItems` will only run if `numbers` changes

TL;DR "use this callback fn if these args change"

useCallback = returns the fn
useMemo = returns the fn value

use case: if creating a function is really really slow...prob not going to run into this...this is more about referential equality like a useEffect hook above (aka "when do I actually care about when this function has changed? Does something else depend on it? ex: its passed as a prop, its used within a hook as a dependency, etc")

# useMemo
memoization - cache a value so you dont have to calculate it everytime
* `useMemo()` - cache the value, dont run the fn unless something changes

ex:
```js
export default function app() {
  const [number, setNumber] = useState(0);
  const [dark, setDark] = useState(false);
  const doubleNumber = slowFunction(number);

  const styles = {
    color: dark ? 'white' : 'black'
  };

  return (
    <>
      <input type="number" value={number} onChange={e => setNumber(parseInt(e.target.value))} />
      <button onClick={() => setDark(prevDark => !prevDark)}>change theme</button>
      <div style={styles}>{doubleNumber}</div>
    </>
  )
}

function slowFunction(num) {
  for(let i = 0; i <=100000000000; i++) {}
  return num * 2;
}
```

^ so here, we have a slow ass fn. this slow ass fn will re-run every time we have a state change in <App>, ex: click the button to toggle the styles, we still call the slow ass fn to calculate the number displayed. no reason to do that, since that slow ass fn only depends on `number` state, not `dark` state, so we can wrap it in `useMemo()` hook so we fire off the fn only if the `number` state changes:

```js
  const doubleNumber = useMemo(() => {
    slowFunction(number);
  }, [number])
```
otherwise we return the same value.

tl;dr: useMemo = dont calculate the value (run this fn) unless this thing here changes

but remember: this saves the value of the memoized fn in memory, so overtime this could get big...only use this hook whenever you're calculating something expensive.

also: referential equality

value vs. reference = try to compare 2 diff obj/arrays in JS, it does it based on where it's stored in memory. 'referential' refers to the place in memory where its stored, so you could have this:

```js
const obj1 = {
  name: 'David'
};
const obj2 = {
  name: 'David'
};

console.log(obj1 === obj2); //false
```
how this relates to useMemo:

```js
const styles = {
  color: dark ? 'white' : 'black'
};
//say we wanna run a useEffect whenever the styles change...
useEffect(() => {
  console.log('theme changed');
}, [styles])
```
^ runs everytime because it builds a new object and though they have the same value, the referential equality is NOT the same....we can fix this via `useMemo()`

```js
const styles = useMemo(() => ({
  color: dark ? 'white' : 'black'
}, [dark]);

useEffect(() => {
  console.log('theme changed');
}, [styles])
```

#### useMemo use cases
1. wanna not re-compute an expensive fn
2. when you wanna make sure the reference of an object/array is the exact same as last time


# context
share state b/w components in your react tree...helps solve prop drilling

NOTE: before using context as a way to avoid prop-drilling, consider component composition FIRST

component composition helps flatten the tree, so to speak.


1. first create your context...here you could provide default values if you wanted but we're not doin that
```js
const CountContext = React.createContext();
```

2. allow the context to be consumed via a provider. the provider contains the state to create and change, necessary for your components to use and access:
```js
function CountProvider(props) {
  const [count, setCount] = React.useState(0);
  const value = [count, setCount];
  return <CountContext.Provider value={value} {...props} />;
}
```

3. wrap your components in the provider so they may access the context:
```js
<CountProvider>
  <CountDisplay />
  <Counter />
</CountProvider>
```

4. use the context within components:
```js
function CountDisplay() {
  const [count] = React.useContext(CountContext);
  return <div>{`The current count is ${count}`}</div>
}
```

remember, you can skip array elements if you so choose:
```js
  const [,setCount] = React.useContext(CountContext);
```

TL;DR
1. create context
2. create provider component with state values, allowing consumers to use them
3. wrap components in provider component
4. components can use context now.

note: you can ready attach state on a provider like:
```js
      <CountProvider value={count: 0}>
        <CountDisplay />
      </CountProvider>
```
but having it within its own component allows you run hooks internal state...can be much more useful:
```js
function CountProvider(props) {
  const [count, setCount] = React.useState(0);
  const value = [count, setCount];
  return <CountContext.Provider value={value} {...props} />;
}

<CountProvider>
  <CountDisplay />
</CountProvider>
```



Problem: imagine your component that uses the context is used outside of its provider, it would error out and it'd be hard to debug:
```js
<CountProvider>
  <CountDisplay />
</CountProvider>
  <Counter />
```
^ you could create a custom hook to throw an error if not called within provider.

the custom hook is just a function that acts as a proxy for the the context, so context doesnt need to worry about error handling:
```js
function useCount() {
  const context = React.useContext(CountContext);
  if (!context) {
    throw new Error(' ust be rendered within CountContext Provider');
  }
  return context;
}


function CountDisplay() {
  const [count] = useCount();
  return <div>{`The current count is ${count}`}</div>
}

function App() {
  return (
    <div>
      <CountProvider>
        <CountDisplay />
      </CountProvider>
        <Counter />
    </div>
  )
}
```

another use case for a custom hook!!!

interesting: think of error handling more...specifically in terms of hooks

for extension: have component composition as flat as possible


# useLayoutEffect
`useLayoutEffect` - use only if you need to mutate the DOM, otherwise `useEffect` OR you need to perform measurements

99% of the time, useEffect is a better choice

* useLayoutEffect: runs immediately after react does all DOM mutations....we run this before the browser paints the screen.
* useEffect - runs after component render

ex: imagine you want to programatically scroll to a specific pixel point in the DOM

# useDebugValue
this is used to surface up custom hook values (of your choice) within react devtools upon selecting the component....this way you may easily see what your internal state or whatever it is you want to know about your custom hook from the console.

ex
```js
function useMedia(query, initialState = false) {
  const [state, setState] = React.useState(initialState)
  React.useDebugValue({query, state});

  React.useEffect(() => {
    let mounted = true
    const mql = window.matchMedia(query)
    function onChange() {
      if (!mounted) {
        return
      }
      setState(Boolean(mql.matches))
    }

    mql.addListener(onChange)
    setState(mql.matches)

    return () => {
      mounted = false
      mql.removeListener(onChange)
    }
  }, [query])

  return state
}

function Box() {
  const isBig = useMedia('(min-width: 1000px)')
  const isMedium = useMedia('(max-width: 999px) and (min-width: 700px)')
  const isSmall = useMedia('(max-width: 699px)')
  const color = isBig ? 'green' : isMedium ? 'yellow' : isSmall ? 'red' : null

  return <div style={{width: 200, height: 200, backgroundColor: color}} />
}

function App() {
  return <Box />
}
```

