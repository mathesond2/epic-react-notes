
```js
  function login(formData) {
    console.log('login', formData)
  }
  //NOTICE THAT THERE ARE NO PARAMS!!!!
     <LoginForm
              onSubmit={login}
```

```js
function LoginForm({onSubmit, submitButton}) {
  function handleSubmit(event) {
    event.preventDefault()
    const {username, password} = event.target.elements

    onSubmit({
      username: username.value,
      password: password.value,
    })
  }

    return (
    <form
      css={{
        display: 'flex',
        flexDirection: 'column',
        alignItems: 'stretch',
        '> div': {
          margin: '10px auto',
          width: '100%',
          maxWidth: '300px',
        },
      }}
      onSubmit={handleSubmit}
    >
```

**You might consider making the network request in the event handler. In general I
recommend to do all your side effects inside the `useEffect`. This is because in
the event handler you don't have any possibility to prevent race conditions, or
to implement any cancellation mechanism.**

cool: creating a client:
```js
function client(endpoint, customConfig = {}) {
  const config = {
    method: 'GET',
    ...customConfig,
  };

  const fullURL = `${process.env.REACT_APP_API_URL}/${endpoint}`;

  return window.fetch(fullURL, config)
    .then(response => response.json());
}

export { client }
```


`window.fetch()` wont reject your promise unless the network request failed (aka you couldnt even talk to the server)...otherwise, it will resolve...what it will do instead is change the `ok` property to `false`.

so say you get a 500 err back, you'll still get a response, but it will include a `res.ok` of `false`
```js
  return window.fetch(fullURL, config)
    .then(async (response) => {
      const data = await response.json();
      if (response.ok) {
        return data;
      } else {
        return Promise.reject(data);
      }
    });
```

look at this `useAsync` hook:
```js
function useSafeDispatch(dispatch) {
  const mounted = React.useRef(false)
  //whenever painted, run useLayoutEffect to be true, being false on cleanup
  React.useLayoutEffect(() => {
    mounted.current = true
    return () => (mounted.current = false)
  }, [])

  //return a memoized dispatch fn if mounted
  return React.useCallback(
    (...args) => (mounted.current ? dispatch(...args) : void 0),
    [dispatch],
  )
}


// Example usage:
// const {data, error, status, run} = useAsync()
// React.useEffect(() => {
//   run(fetchPokemon(pokemonName))
// }, [pokemonName, run])
const defaultInitialState = {status: 'idle', data: null, error: null}

function useAsync(initialState) {
  //create a ref for initial state
  const initialStateRef = React.useRef({
    ...defaultInitialState,
    ...initialState,
  })

  //create a way to update state via useReducer, destructuring the state object values...we can use useReducer b/c these values will likely update at the same time (ex: 'endpoint call, status changes, data changes, maybe an err, etc').
  const [{status, data, error}, setState] = React.useReducer(
    (s, a) => ({...s, ...a}),
    initialStateRef.current,
  )

  //safely setState using the memoized fn from `useSafeDispatch` hook
  const safeSetState = useSafeDispatch(setState)

  //use `useCallback` hook to only re-render the fn if safeSetState changes...this is useful b/c we'll be exporting this fn and it could be used in a variety of ways (ex passed as a prop).
  const setData = React.useCallback(
    data => safeSetState({data, status: 'resolved'}),
    [safeSetState],
  )
  const setError = React.useCallback(
    error => safeSetState({error, status: 'rejected'}),
    [safeSetState],
  )
  const reset = React.useCallback(
    () => safeSetState(initialStateRef.current),
    [safeSetState],
  )
  //check if arg is promise, do err handling if not, otherwise update state, run the promise, set the data in state and return it, and same if err...use `useCallback` to ensure that this fn is only updated when the deps change and to ensure that data is not stale. and remember, these consts (ex: setError) are updated via `safeSetState` which is the memoized fn from useSafeDispatch hook

  const run = React.useCallback(
    promise => {
      if (!promise || !promise.then) {
        throw new Error(
          `The argument passed to useAsync().run must be a promise. Maybe a function that's passed isn't returning anything?`,
        )
      }
      safeSetState({status: 'pending'})
      return promise.then(
        data => {
          setData(data)
          return data
        },
        error => {
          setError(error)
          return Promise.reject(error)
        },
      )
    },
    [safeSetState, setData, setError],
  )

  return {
    // using the same names that react-query uses for convenience
    //DM: assign values
    isIdle: status === 'idle',
    isLoading: status === 'pending',
    isError: status === 'rejected',
    isSuccess: status === 'resolved',

    setData,
    setError,
    error,
    status,
    data,
    run,
    reset,
  }
}

export {useAsync}
```


# authentication
user/password combo is a common way to do auth, but a user doesnt wanna have to submit a password everytime they request some data. we wanna be able to have a user log into the app, we store the info and then a user can leave and come back and get whatever data they need.

we could send that stored user/pass info on every request. however a better way to do this is through generating a token each time a user signs in. this token is just a string, which we'll send on every req. in the case the token is lost or stolen, we can invalidate the token and just give the user a new token upon re-authenticating...this way a user doesnt have to change their password.

so in our client, we'll need the user's token within the `headers` object for every req:
```js
window.fetch('http://example.com/pets', {
  headers: {
    Authorization: `Bearer ${token}`,
  },
})
```

a token can be anything that identifies a user but a common way is via JSON web tokens (JWTs).

use an auth provider for this stuff:
1. call some API to retrieve a token
2. if there is a token, send it along w the req to make:
```javascript
const token = await authProvider.getToken()
const headers = {
  Authorization: token ? `Bearer ${token}` : undefined,
}
window.fetch('http://example.com/pets', {headers})
```

the `useAsync` custom hook seems super useful, pretty complex initially to look into but completely isolates all API call sideEffects from the component...or does it?

well yea bc you're still defining the endpoint you wanna hit and then also the state you want to be saved, you're just doing so within the context of the custom hook, to leverage its capabilities re: storing state and producing side effects, ex:
```js
React.useEffect(() => {
  run(getUser());
}, [run]);

//calls endpoints here:
async function getUser() {
  let user = null;
  const token = await auth.getToken();
  if (token) {
  const data = await client('me', token)
  user = data.user;
  }

  return user;
}
```

# cache mgmt
the idea is to separate state out into 2 areas:
* UI State - 'dropdown is open/closed'
* server state - data from endpoint, isloading, iserr, etc

theres a lib called react-query which handles server state completely, including calling endpoints.

interesting...optional chaining exists in JS alone.

this is interesting too:
```js
const loadingBooks = Array.from({ length: 10 }, (v, index) => ({
  id: `loading-book-${index}`,
  ...loadingBook,
}))
```

not even fucking w this module

# context
we could always remove the arg, import the context and use the `useContext` react hook to grab a value from context like this:

```js
function useCreateListItem(options, user) {
  return useMutation(
    ({ bookId }) => client(`list-items`, { data: { bookId }, token: user.token }),
    { ...defaultMutationOptions, ...options },
  )
}
```
to
```js
import { AuthContext } from 'context/auth-context'

function useCreateListItem(options) {
  const { user } = React.useContext(AuthContext);
  return useMutation(
    ({ bookId }) => client(`list-items`, { data: { bookId }, token: user.token }),
    { ...defaultMutationOptions, ...options },
  )
}
```

but what if we could just get the data without having to worry about worry it comes from, not having to import the context use the hook? also if we dont have the component within the context, we'll get an err...what can we do instead?

`context/auth-context.js`
```js
import React from 'react'
const AuthContext = React.createContext()

function useAuth() {
  const context = React.useContext(AuthContext)
  if (context === undefined) {
    throw new Error('useAuth must be used within AuthContext Provider');
  }
  return context
}

export {AuthContext, useAuth}
```

we can use the context from here and now we have a hook we can use to plugin into anytime we need to use the context..this gives us a few benefits:
- instant err checking
- dont have to import any context or useContext hook
- this hook exists as a proxy to our actual context, so we could do anything we really want within here.

**TLDR: create a custom hook for using context so our consumers dont need to know where we get the data from and having a proxy layer b/w the consumer and the context allowing us to do err checking and any additional behavior.**

now we could take this one BIG step further...look in our app currently:

`app.js`
```js
function App() {
  const {
    data: user,
    error,
    isLoading,
    isIdle,
    isError,
    isSuccess,
    run,
    setData,
  } = useAsync()

  React.useEffect(() => {
    run(getUser())
  }, [run])

  const login = form => auth.login(form).then(user => setData(user))
  const register = form => auth.register(form).then(user => setData(user))
  const logout = () => {
    auth.logout()
    setData(null)
  }

  if (isLoading || isIdle) {
    return <FullPageSpinner />
  }

  if (isError) {
    return <FullPageErrorFallback error={error} />
  }

  if (isSuccess) {
    const props = { user, login, register, logout }
    return (
      <AuthContext.Provider value={props}>
        {user ? (
          <Router>
            <AuthenticatedApp />
          </Router>
        ) : (
          <UnauthenticatedApp />
        )}
      </AuthContext.Provider>
    )
  }
```

what if we could encapsulate everything auth-based into our context/auth-context.js file? this way our App component is dirt simple...take a look at the result:
```js
import * as React from 'react'
import {BrowserRouter as Router} from 'react-router-dom'
import {useAuth} from './context/auth-context'
import {AuthenticatedApp} from './authenticated-app'
import {UnauthenticatedApp} from './unauthenticated-app'

function App() {
  const {user} = useAuth()
  return user ? (
    <Router>
      <AuthenticatedApp />
    </Router>
  ) : (
    <UnauthenticatedApp />
  )
}

export {App}
```
so we can use `useAuth` like we did before, but we'll need to wrap our App component in the provider `AuthProvider` that we'll be creating:

index.js
```js
  ReactDOM.render(
    <ReactQueryConfigProvider config={queryConfig}>
      <AuthProvider>
        <App />
      </AuthProvider>
    </ReactQueryConfigProvider>,
    document.getElementById('root'),
  )
```

sounds good, but we'll have to create `AuthProvider` within `auth-context.js`:
```js
import {jsx} from '@emotion/core'

import * as React from 'react'
import {queryCache} from 'react-query'
import * as auth from 'auth-provider'
import {client} from 'utils/api-client'
import {useAsync} from 'utils/hooks'
import {FullPageSpinner, FullPageErrorFallback} from 'components/lib'

async function getUser() {
  let user = null

  const token = await auth.getToken()
  if (token) {
    const data = await client('me', {token})
    user = data.user
  }

  return user
}

const AuthContext = React.createContext()
AuthContext.displayName = 'AuthContext'

function AuthProvider(props) {
  const {
    data: user,
    error,
    isLoading,
    isIdle,
    isError,
    isSuccess,
    run,
    setData,
    status,
  } = useAsync()

  React.useEffect(() => {
    run(getUser())
  }, [run])

  const login = form => auth.login(form).then(user => setData(user))
  const register = form => auth.register(form).then(user => setData(user))
  const logout = () => {
    auth.logout()
    queryCache.clear()
    setData(null)
  }

  if (isLoading || isIdle) {
    return <FullPageSpinner />
  }

  if (isError) {
    return <FullPageErrorFallback error={error} />
  }

  if (isSuccess) {
    const value = {user, login, register, logout}
    return <AuthContext.Provider value={value} {...props} />
  }

  throw new Error(`Unhandled status: ${status}`)
}
```

so lets rundown on everything that happens here:
* use `getUser()` in order to get the token if it does exists (via `auth.getToken()`) and then call our `client` fn to call the fetch request with auth token and necessary http verbs, headers, etc
```js
async function getUser() {
  let user = null

  const token = await auth.getToken()
  if (token) {
    const data = await client('me', {token})
    user = data.user
  }

  return user
}
```
* we'll then use the `useAsync` hook to use the `run()` fn to call `getUser()` within it (via a useEffect hook on initial render )...using `run()` allows us to then also get the necessary side effect variables like:
```js
function AuthProvider(props) {
  const {
    data: user,
    error,
    isLoading,
    isIdle,
    isError,
    isSuccess,
    run,
    setData,
    status,
  } = useAsync()

  React.useEffect(() => {
    run(getUser())
  }, [run])
```
^ notice we have `run` here in the dep array order to make sure we dont have stale data within the `useEffect`

we then have the onClick handlers also within the provider that use `setData` from our `useAsync` hook:
```js
  const login = form => auth.login(form).then(user => setData(user))
  const register = form => auth.register(form).then(user => setData(user))
  const logout = () => {
    auth.logout()
    queryCache.clear()
    setData(null)
  }
```
these fns will then be attached to the Provider that is returned upon success:
```js
  const login = form => auth.login(form).then(user => setData(user))
  const register = form => auth.register(form).then(user => setData(user))
  const logout = () => {
    auth.logout()
    queryCache.clear()
    setData(null)
  }

  if (isLoading || isIdle) {
    return <FullPageSpinner />
  }

  if (isError) {
    return <FullPageErrorFallback error={error} />
  }

  if (isSuccess) {
    const value = {user, login, register, logout}
    return <AuthContext.Provider value={value} {...props} />
  }
```

remember, since these fns are exported on the provider, we can access them using our `useAuth()` hook  (our proxy for the context!) we've creted in the same file:
```js
function useAuth() {
  const context = React.useContext(AuthContext)
  if (context === undefined) {
    throw new Error(`useAuth must be used within a AuthProvider`)
  }
  return context
}
```
used like this:
```js
function UnauthenticatedApp() {
  const {login, register} = useAuth()
  return (
    <div>
      <div>
        <Modal>
          <ModalOpenButton>
            <Button variant="primary">Login</Button>
          </ModalOpenButton>
          <ModalContents aria-label="Login form" title="Login">
            <LoginForm
              onSubmit={login}
              submitButton={<Button variant="primary">Login</Button>}
            />
          </ModalContents>
        </Modal>
        <Modal>
          <ModalOpenButton>
            <Button variant="secondary">Register</Button>
          </ModalOpenButton>
          <ModalContents aria-label="Registration form" title="Register">
            <LoginForm
              onSubmit={register}
              submitButton={<Button variant="secondary">Register</Button>}
            />
          </ModalContents>
```

this is great because we have only 1 file to access this context, which is the same file that creates it. our related functionality is encapsulated within a provider in this file to be exported and wrapped the component which needs it (`<App/>`) in addition to the `useAuth` hook which acts as proxy for the context and can be used at will in components that need it


## create a useClient hook
so currently we are using the `useAuth()` custom hook in a few diff fns in our app to get the `user` value from context to then pass it over to our `client()` fn in order to get the `user.token` value:
```js
function useListItems() {
  const {user} = useAuth()
  const {data} = useQuery({
    queryKey: 'list-items',
    queryFn: () =>
      client(`list-items`, {token: user.token}).then(data => data.listItems),
    config: {
      onSuccess: async listItems => {
        for (const listItem of listItems) {
          setQueryDataForBook(listItem.book)
        }
      },
    },
  })
  return data ?? []
}

function useCreateListItem(options) {
  const {user} = useAuth()
  return useMutation(
    ({bookId}) => client(`list-items`, {data: {bookId}, token: user.token}),
    {...defaultMutationOptions, ...options},
  )
}
//etc etc
```

we could create a custom hook that returns an authenticated `client` fn so the associated fns dont have to worry about our context or auth'ing requests. the fns should only be worried about calling the endpoints, thats it
```js
function useClient() {
  const {
    user: {token},
  } = useAuth()
  return React.useCallback(
    (endpoint, config) => client(endpoint, {...config, token}),
    [token],
  )
}
```
we memoize this fn using `useCallback` in the case that someone wants to use this returned fn within a dependency list (ex: within a `useEffect` hook)..just a nice thing to do inside the fn to 'future proof'.. how it could be used.

lets see how its used
```js
function useCreateListItem(options) {
  const client = useClient()
  return useMutation(({bookId}) => client(`list-items`, {data: {bookId}}), {
    ...defaultMutationOptions,
    ...options,
  })
}
```

so lets actually chat about whats happening in the `useClient` hook:
* grab `token` from `user` object off our context using the `useAuth` context consumer custom hook.
* return a memoized fn that only is returned if our value of `token` changes.

we then call `useClient()` in `const client`, const client = a fn to be called with params. `token` is already there in that fn via a closure. we then call `client()` within another hook.

# compound components
how to make components more flexible and not constantly passing conditional props (ex: `isAbove`, `contentBelow`, etc)? what if requirements change and you keep adding conditional props over and over...sound familiar?

huge maintanence overhead, more documentation you have to write to keep up with more use cases, implementation complexity in general (and bugs that sneak up from this)...also the API complexity for the component ('how tf does this work?')

think: inversion of control aka 'I give you the ability to put your use cases in my work instead of baking your use case into my specific code'...ex:
```jsx
<Modal>
  <ModalOpenButton>
    <button>Open Modal</button>
  </ModalOpenButton>
  <ModalContents aria-label="Modal label (for screen readers)">
    <ModalDismissButton>
      <button>Close Modal</button>
    </ModalDismissButton>
    <h3>Modal title</h3>
    <div>Some great contents of the modal</div>
  </ModalContents>
</Modal>
```
from
```jsx
<LoginFormModal
  onSubmit={handleSubmit}
  modalTitle="Modal title"
  modalLabelText="Modal label (for screen readers)"
  submitButton={<button>Submit form</button>}
  openButton={<button>Open Modal</button>}
/>
```

/////////////

so in the exercise we want to build the individual components in a flexible way where we can mix/match as we wish, and use context to allow for shared state regardless of where the components are used:
```js
import {jsx} from '@emotion/core'

import * as React from 'react'
import {Dialog} from './lib'

const ModalContext = React.createContext()

function Modal(props) {
  const [isOpen, setIsOpen] = React.useState(false)

  return <ModalContext.Provider value={[isOpen, setIsOpen]} {...props} />
}

function ModalDismissButton({children: child}) {
  const [, setIsOpen] = React.useContext(ModalContext)
  return React.cloneElement(child, {
    onClick: () => setIsOpen(false),
  })
}

function ModalOpenButton({children: child}) {
  const [, setIsOpen] = React.useContext(ModalContext)
  return React.cloneElement(child, {
    onClick: () => setIsOpen(true),
  })
}

function ModalContents(props) {
  const [isOpen, setIsOpen] = React.useContext(ModalContext)
  return (
    <Dialog isOpen={isOpen} onDismiss={() => setIsOpen(false)} {...props} />
  )
}

export {Modal, ModalDismissButton, ModalOpenButton, ModalContents}
```

1. what is `React.cloneElement` doing?
you can use `React.cloneElement` when you wish to modify children components' props from the parent (aka the parent can create a clone kid with the props it wished for, rather than got aka 'another little brother who skated').
```js
  return React.cloneElement(child, {
    onClick: () => setIsOpen(true),
  })
```
we clone the child and give it an onClick fn, this way the child component can be anything and it will still get this prop from the parent.

2. remember the optional syntax here: `const [, setIsOpen] = React.useContext(ModalContext)`

3. this:
```js
function ModalOpenButton({children: child}) {
  const [, setIsOpen] = React.useContext(ModalContext)
  return React.cloneElement(child, {
```

we alias `children` to the name `child` ...important bc `children` namespace is not to be messed w.



now look at how flexible we've made this component:
```js
const circleDismissButton = (
  <div css={{display: 'flex', justifyContent: 'flex-end'}}>
    <ModalDismissButton>
      <CircleButton>
        <VisuallyHidden>Close</VisuallyHidden>
        <span aria-hidden>×</span>
      </CircleButton>
    </ModalDismissButton>
  </div>
)

<Modal>
  <ModalOpenButton>
    <Button variant="primary">Login</Button>
  </ModalOpenButton>
  <ModalContents aria-label="Login form">
    {circleDismissButton}
    <h3 css={{textAlign: 'center', fontSize: '2em'}}>Login</h3>
    <LoginForm
      onSubmit={login}
      submitButton={<Button variant="primary">Login</Button>}
    />
  </ModalContents>
</Modal>
<Modal>
  <ModalOpenButton>
    <Button variant="secondary">Register</Button>
  </ModalOpenButton>
  <ModalContents aria-label="Registration form">
    {circleDismissButton}
    <h3 css={{textAlign: 'center', fontSize: '2em'}}>Register</h3>
    <LoginForm
      onSubmit={register}
      submitButton={<Button variant="secondary">Register</Button>}
    />
  </ModalContents>
</Modal>
```
so all of these associated components share state via their own context, regardless how they are wrapped/used (ex look at `circleDismissButton`), and the consuming components dont need to know anything about it. Also the users of our component can combine and compose associated `Modal` components however they'd like, giving an **inversion of control** to the component user.


now imagine if we want this component to be even more flexible...what if we want it to be so a user of the modal can either have our stock `ModalContents` content OR they just bring their own?

aka go from this:
```js
<Modal>
  <ModalOpenButton>
    <Button variant="primary">Login</Button>
  </ModalOpenButton>
  <ModalContents aria-label="Login form">
    {circleDismissButton}
    <h3 css={{textAlign: 'center', fontSize: '2em'}}>Login</h3>
    <LoginForm
      onSubmit={login}
      submitButton={<Button variant="primary">Login</Button>}
    />
  </ModalContents>
</Modal>
```
to
```js
<Modal>
  <ModalOpenButton>
    <Button variant="primary">Login</Button>
  </ModalOpenButton>
  <ModalContents aria-label="Login form" title="Login">
    <LoginForm
      onSubmit={login}
      submitButton={<Button variant="primary">Login</Button>}
    />
  </ModalContents>
</Modal>
```
so basically baking in this:
```js
{circleDismissButton}
    <h3 css={{textAlign: 'center', fontSize: '2em'}}>Login</h3>
```

here's how we do it:
```js
function ModalContentsBase(props) {
  const [isOpen, setIsOpen] = React.useContext(ModalContext)
  return (
    <Dialog isOpen={isOpen} onDismiss={() => setIsOpen(false)} {...props} />
  )
}

function ModalContents({title, children, ...props}) {
  return (
    <ModalContentsBase {...props}>
      <div css={{display: 'flex', justifyContent: 'flex-end'}}>
        <ModalDismissButton>
          <CircleButton>
            <VisuallyHidden>Close</VisuallyHidden>
            <span aria-hidden>×</span>
          </CircleButton>
        </ModalDismissButton>
      </div>
      <h3 css={{textAlign: 'center', fontSize: '2em'}}>{title}</h3>
      {children}
    </ModalContentsBase>
  )
}
```

`ModalContentsBase` is to be used for folks that want a full customization, otherwise we can use `ModalContents` for stock components.

interesting: `ModalContents({title, children, ...props})` destructuring `title` and `children` while spreading props