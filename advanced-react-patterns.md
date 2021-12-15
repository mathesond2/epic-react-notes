## compound components
look at this
```js
<select>
  <option value="1">Option 1</option>
  <option value="2">Option 2</option>
</select>
```
^ select is the top-level el and option is the child. they share state implicitly (select holds the state to show what option is selected, and options show what options are available).

```js
<CustomSelect
  options={[
    {value: '1', display: 'Option 1'},
    {value: '2', display: 'Option 2'},
  ]}
/>
```
Imagine if you wish to do more on a select element than just show the option selected:
  * what if you want `display` to change based on whether it's selected
  * what if I, for some reason, wish to supply additional attrs on the <option> element rendered?

we can make the component more extensible using the compound component pattern:
```js
function Toggle({children}) {
  const [on, setOn] = React.useState(false)
  const toggle = () => setOn(!on)
  return React.Children.map(children, child =>
    React.cloneElement(child, {on, toggle}),
  )
}

function ToggleOn({on, children}) {
  return on ? children : null
}

function ToggleOff({on, children}) {
  return on ? null : children
}

function ToggleButton({on, toggle, ...props}) {
  return <Switch on={on} onClick={toggle} {...props} />
}

function App() {
  return (
    <div>
      <Toggle>
        <ToggleOn>The button is on</ToggleOn>
        <ToggleOff>The button is off</ToggleOff>
        <ToggleButton />
      </Toggle>
    </div>
  )
}
```
^ this overcomplicates stuff like hell. I know. But, the allows us to extend the `Toggle` component by being able to share state with its children. `React.Children.map()` allows us to iterate on the children within `Toggle`, and pass it both state and a function to use.

kind of complicated.

**TL;DR make a clone of all the children and pass them all the props they need to implicitly share state between component and children, this way you may add/extend remove components as necessary**

you can also run into the issue where nested elements arent able to be found by react, so state may not be implicitly shared:
```js
function App() {
  return (
    <div>
      <Toggle>
        <ToggleOn>The button is on</ToggleOn>
        <ToggleOff>The button is off</ToggleOff>
        <div>
          <ToggleButton />
        </div>
      </Toggle>
    </div>
  )
}
```
^ in this case ToggleButton is fucked. we can make this state more extensible by using context in order to allow state to be implicitly shared regardless of where a componet happens to be in the jsx tree:
```js
//1. create the context
const ToggleContext = React.createContext()

function Toggle({ onToggle, children }) {
  const [on, setOn] = React.useState(false)
  const toggle = () => setOn(!on)
  //wrap children in context provider with state and state updater fn passed as value
  return (
    <ToggleContext.Provider value={{ on, toggle }} >
      {children}
    </ToggleContext.Provider>
  )
}
//create a custom hook as a context proxy, in order not to expose your components directly to the context
function useToggle() {
  return React.useContext(ToggleContext)
}

//use the custom hook
function ToggleOn({ children }) {
  const { on } = useToggle()
  return on ? children : null
}

function ToggleOff({ children }) {
  const { on } = useToggle()
  return on ? null : children
}

function ToggleButton({ ...props }) {
  const { on, toggle } = useToggle()
  return <Switch on={on} onClick={toggle} {...props} />
}

function App() {
  return (
    <div>
      <Toggle>
        <ToggleOn>The button is on</ToggleOn>
        <ToggleOff>The button is off</ToggleOff>
        <div>
          <ToggleButton />
        </div>
      </Toggle>
    </div>
  )
}
```



what was the purpose of creating the custom hook as an intermediary b/w the consumers of the context and the context itself?? For one, you can do error checking:
```js
function useToggle() {
  const context = React.useContext(ToggleContext);
  if (!context) {
    throw new Error('must be rendered with ToggleContext Provider!');
  }
  return context;
}
```

# patterns with custom hooks



## prop collections
idea: take the common use cases for our hook (and/or components) and make objects of props that people can simply spread across the UI they render
```js
function useToggle() {
  const [on, setOn] = React.useState(false)
  const toggle = () => setOn(!on)

  // 🐨 Add a property called `togglerProps`. It should be an object that has
  // `aria-pressed` and `onClick` properties.
  const togglerProps = { 'aria-pressed': on, onClick: toggle }
  return { on, togglerProps }
}

function App() {
  const { on, togglerProps } = useToggle()
  return (
    <div>
      <Switch on={on} {...togglerProps} />
      <hr />
      <button aria-label="custom-button" {...togglerProps}>
        {on ? 'on' : 'off'}
      </button>
    </div>
  )
}
```

## prop getters
so above you are calling the onClick fn within the custom hook `useToggle()` via `togglerProps`, but what if you wanted to add the flexibility for a user of `button` to add their on onClick handler?

```js
<button
  aria-label="custom-button"
  {...togglerProps}
  onClick={() => console.info('onButtonClick')}
>
```

lets enable that by creating a fn, `getTogglerProps()` which accepts all props given, spreads em, and then for our onClick case, checks to see if one already exists, and includes it if so:
```js
function useToggle() {
  const [on, setOn] = React.useState(false)
  const toggle = () => setOn(!on)

  const getTogglerProps = ({ onClick, ...props } = {}) => ({
    'aria-pressed': on,
    onClick: () => {
      onClick && onClick()
      toggle()
    },
    ...props,
  })

  return { on, getTogglerProps }
}

function App() {
  const { on, getTogglerProps } = useToggle()
  return (
    <div>
      <Switch {...getTogglerProps({ on })} />
      <hr />
      <button
        {...getTogglerProps({
          'aria-label': 'custom-button',
          onClick: () => console.log('onButtonClick'),
          id: 'custom-id'
        })}
      >
        {on ? 'on' : 'off'}
      </button>
    </div>
  )
}
```
^ this PROP GETTER is more flexible than our PROP COLLECTIONS as it allows for users of the button to add what they need for their use cases.


# state reducer pattern
_A pattern to use in custom hooks to enhance its power and flexibility_

problem: component is used in many different contexts, and feature requests are constantly asking for this component to do new things. we could do like has been done at Bitly where we keep adding new props and conditional logic to the component to allow it to suffice all these additional use cases OR we could do something else...

inversion of control - allows for the author of the API to allow the user of the API to control how things work.

ex: kent had made a typeahead component with so that when an end user selects an item in the dropdown, the `isOpen` state should be set to false and the menu should close. someone was building a multi-select with this component and wanted to keep the menu open so they could continue selecting options. By inverting control of state updates with this new pattern, he can enable this use case and any other use case ppl could want whenever they wanna change how this component operates internally.

concept:
1. End user does an action
2. Dev calls dispatch
3. Hook determines the necessary changes
4. Hook calls dev's code for further changes 👈 this is the inversion of control part
5. Hook makes the state changes

ex:

we
```js
// 🐨 add a new option called `reducer` that defaults to `toggleReducer`
function useToggle({initialOn = false} = {}) {
  const {current: initialState} = React.useRef({on: initialOn})
  // 🐨 instead of passing `toggleReducer` here, pass the `reducer` that's
  // provided as an option
  // ... and that's it!
  const [state, dispatch] = React.useReducer(toggleReducer, initialState)
  const {on} = state

  const toggle = () => dispatch({type: 'toggle'})
  const reset = () => dispatch({type: 'reset', initialState})

  function getTogglerProps({onClick, ...props} = {}) {
    return {
      'aria-pressed': on,
      onClick: callAll(onClick, toggle),
      ...props,
    }
  }

  function getResetterProps({onClick, ...props} = {}) {
    return {
      onClick: callAll(onClick, reset),
      ...props,
    }
  }

  return {
    on,
    reset,
    toggle,
    getTogglerProps,
    getResetterProps,
  }
}
```
to
```js
function useToggle({initialOn = false, reducer = toggleReducer} = {}) {
  const {current: initialState} = React.useRef({on: initialOn})
  // 🐨 instead of passing `toggleReducer` here, pass the `reducer` that's
  // provided as an option
  // ... and that's it! Don't forget to check the 💯 extra credit!
  const [state, dispatch] = React.useReducer(toggleReducer, initialState)
  const {on} = state

  const toggle = () => dispatch({type: 'toggle'})
  const reset = () => dispatch({type: 'reset', initialState})

  function getTogglerProps({onClick, ...props} = {}) {
    return {
      'aria-pressed': on,
      onClick: callAll(onClick, toggle),
      ...props,
    }
  }

  function getResetterProps({onClick, ...props} = {}) {
    return {
      onClick: callAll(onClick, reset),
      ...props,
    }
  }

  return {
    on,
    reset,
    toggle,
    getTogglerProps,
    getResetterProps,
  }
}
```

this allows the custom hook to use either the default reducer (`toggleReducer`) or one supplied by the user of the component, hence inverting the control externally.

1. extra credit:
now in our example, we're 'giving away our control' of always using `toggleReducer` in favor of allowing other reducers, like `toggleStateReducer` to be used in the custom hook. `toggleStateReducer` is going to have to be v similar to `toggleReducer` in terms of its action types, but reducers could be really complex. we can make this easier for our users of our custom hook to modify, by just giving them the reducer and allowing them to make modifications on top of it:
```js
  // function toggleStateReducer(state, action) {
  //   switch (action.type) {
  //     case 'toggle': {
  //       if (clickedTooMuch) {
  //         return { on: state.on }
  //       }
  //       return { on: !state.on }
  //     }
  //     case 'reset': {
  //       return { on: false }
  //     }
  //     default: {
  //       throw new Error(`Unsupported type: ${action.type}`)
  //     }
  //   }
  // }

  function toggleStateReducer(state, action) {
    if (action.type === 'toggle' && timesClicked >= 4) {
      return { on: state.on }
    }
    return toggleReducer(state, action)
  }
```

when you think of the inversion of control in regards to JS, think of adding "slots" to your functions' internal workings where users can add in functionality as they need to do so.

ex: a filter function - modify the api of the filter fn to allow for users to provide their own filters to the function. This makes the filter function able to account for a ton of new use cases it previously could not have (say for example if it just filtered zeroes from a list initially) AND is a more obvious api (ex: `filter(input, filterNullAndUndefined);`)

^ THIS IS COMPOSITION.

this makes implementations more simple in order not to make abstractions more complicated internally (additional logic, another if statement, etc)...allow users of the fn to change functionality as they wish

inversion of control: "im not modifying my work to fit your needs, but I'll give you a slot to put in your use case"

**TBH I do not understand the control props pattern. I understand its a way to allow props to control the internal state and workings of your component(s) in the same way we have controlled inputs in react, but I do not understand this shit at all.**
