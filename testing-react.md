- "if I were a manual tester, how would I test this?" - when looking at component you want to test.
- "the more test resemble how your software is used, the more confidence they can give you"

check this out:
```
  const [decrement, increment] = div.querySelectorAll('button');
  const message = div.firstChild.querySelector('div')
  const mouseEv = new MouseEvent('click', {
    bubbles: true,
    cancelable: true,
    button: 0,
  });

  expect(message.textContent).toBe('Current count: 0');
  increment.dispatchEvent(mouseEv);
  expect(message.textContent).toBe('Current count: 1');
```
^ here we could just say `increment.click()` but we can use Dispatch event for ANY browser user action (ex: mouseover)...this gives us the capability to test more stuff.


react testing library
```javascript
import {render, fireEvent, screen} from '@testing-library/react'

test('it works', () => {
  const {container} = render(<Example />)
  // container is the div that your component has been mounted onto.

  const input = container.querySelector('input')
  fireEvent.mouseEnter(input) // fires a mouseEnter event on the input

  screen.debug() // logs the current state of the DOM (with syntax highlighting!)
})
```
^ theres no cleanup here, bc the library has an auto-cleanup feature

```js
  const {container} = render(<Counter />)

```

renders component to dom via a div called `container` that is specific to the library.

testing library has a suite of Jest assertions already baked into the project, but you can switch to more specific assertions to get better err msgs via `testing-library/jest-dom`.

avoid implementation details = shit your users dont care about

keep your tests as impelemntation detail free as possible

^ helps so you dont have to refactor your tests if implementation details change.

(ex: going from class components to functional components)

how to avoid them?

**You can tell whether tests
rely on implementation details if they're written in a way that would fail if
the implementation changes**.

For example, what if we wrapped our counter
component in another `div` or swapped our message from a `div` to a `span` or
`p`?

look at this: (implementation details)
```js
import * as React from 'react'
// 🐨 add `screen` to the import here:
import {render, fireEvent} from '@testing-library/react'
import Counter from '../../components/counter'

test('counter increments and decrements when the buttons are clicked', () => {
  const {container} = render(<Counter />)

  //imagine if we add another button
  const [decrement, increment] = container.querySelectorAll('button')
  //imagine if we add a div before or change this element to a span
  const message = container.firstChild.querySelector('div')

  expect(message).toHaveTextContent('Current count: 0')
  fireEvent.click(increment)
  expect(message).toHaveTextContent('Current count: 1')
  fireEvent.click(decrement)
  expect(message).toHaveTextContent('Current count: 0')
})
```


vs this:
```js
import * as React from 'react'
import { render, fireEvent, screen } from '@testing-library/react'
import Counter from '../../components/counter'
import userEvent from '@testing-library/user-event'

test('counter increments and decrements when the buttons are clicked', () => {
  render(<Counter />)
  const increment = screen.getByRole('button', { name: /increment/i })
  const decrement = screen.getByRole('button', { name: /decrement/i })
  const message = screen.getByText(/current count/i)

  expect(message).toHaveTextContent('Current count: 0')
  userEvent.click(increment)
  expect(message).toHaveTextContent('Current count: 1')
  userEvent.click(decrement)
  expect(message).toHaveTextContent('Current count: 0')
})
```
`userEvent` tries to mock out exactly how a user would click something (ex: first mouseover event, then mousedown, then click, mouseout, etc)...this is nice too b/c what if we change the `onClick` event to `onMouseOver`.

defer to userEvent over fireEvent when you can to avoid implementation details as much as possible.

**this^ makes the tests more resilient to changes in the code over time.**

another cool thing about react-testing-library is that it tests with accessibility in mind as well, i.e. what you test is exactly how ALL users interact with your app (ex `getByRole()`)

if your unfamiliar and you dont know exactly what query you should be using, you can use https://testing-playground.com/ to get suggested queries for your element.

# testing forms
mocking fns...mocked fns are known as 'spies' bc we can indirectly test that fns are called rather than just testing the output of something. check out this component.
```js
function Login({onSubmit}) {
  function handleSubmit(event) {
    event.preventDefault()
    const {username, password} = event.target.elements

    onSubmit({
      username: username.value,
      password: password.value,
    })
  }
  return (
    <form onSubmit={handleSubmit}>
      <div>
        <label htmlFor="username-field">Username</label>
        <input id="username-field" name="username" type="text" />
      </div>
      <div>
        <label htmlFor="password-field">Password</label>
        <input id="password-field" name="password" type="password" />
      </div>
      <div>
        <button type="submit">Submit</button>
      </div>
    </form>
  )
}
```

^ we can mock `handleSubmit` fn in our tests and test that its been called w the right params:
```js
test('submitting the form calls onSubmit with username and password', () => {
  const mockFn = jest.fn();
  render(<Login onSubmit={mockFn} />);
  const password = screen.getByLabelText(/password/i);
  const username = screen.getByLabelText(/username/i);
  userEvent.type(username, 'myusername');
  userEvent.type(password, 'mypassword');

  const submit = screen.getByRole('button', { name: /submit/i })
  userEvent.click(submit);
  expect(mockFn).toHaveBeenCalledWith({
    username: 'myusername',
    password: 'mypassword'
  });
})
```

whoa check this out
```js
const mockFn = jest.fn();
  render(<Login onSubmit={mockFn} />);
  screen.debug() //this
```
^ shows what jsx is actually rendered in the terminal window...v cool.

# mocking http requests
`whatwg-fetch` - module polyfills fetch for our jest tests.

it takes a lot of setup to test FE integration w BE, typically a E2E thing. We'll tradeoff some confidence for convenience and for our jest tests, startup a mock server to handle fetch reqs.

its not actually a server, but a req interceptor. we'll use a tool called MSW. ex:
```javascript
// __tests__/fetch.test.js
import * as React from 'react'
import {rest} from 'msw'
import {setupServer} from 'msw/node'
import {render, waitForElementToBeRemoved, screen} from '@testing-library/react'
import {userEvent} from '@testing-library/user-event'
import Fetch from '../fetch'

const server = setupServer(
  rest.get('/greeting', (req, res, ctx) => {
    return res(ctx.json({greeting: 'hello there'}))
  }),
)

beforeAll(() => server.listen())
afterEach(() => server.resetHandlers())
afterAll(() => server.close())

test('loads and displays greeting', async () => {
  render(<Fetch url="/greeting" />)

  userEvent.click(screen.getByText('Load Greeting'))

  await waitForElementToBeRemoved(() => screen.getByText('Loading...'))

  expect(screen.getByRole('heading')).toHaveTextContent('hello there')
  expect(screen.getByRole('button')).toHaveAttribute('disabled')
})
```

`setupServer` allows us to intercept requests to specified endpoints (ex: `'/greeting` but imagine this also for v4_public endpoints as well...)

**"In my applications, I love having a mock server to use during development. It's
often more reliable, works offline, doesn't require a lot of environment setup,
and allows me to start writing UI for APIs that aren't finished yet."**

## preserving your mock endpoints and responses
you can also save these server req handlers in a sep file, going from:
```js
  rest.get('/greeting', (req, res, ctx) => {
    return res(ctx.json({greeting: 'hello there'}))
  }),
```

to:
```js
import {rest} from 'msw'

const delay = process.env.NODE_ENV === 'test' ? 0 : 1500

const handlers = [
  rest.post(
    'https://auth-provider.example.com/api/login',
    async (req, res, ctx) => {
      if (!req.body.password) {
        return res(
          ctx.delay(delay),
          ctx.status(400),
          ctx.json({message: 'password required'}),
        )
      }
      if (!req.body.username) {
        return res(
          ctx.delay(delay),
          ctx.status(400),
          ctx.json({message: 'username required'}),
        )
      }
      return res(ctx.delay(delay), ctx.json({username: req.body.username}))
    },
  ),
]

export {handlers}
```
```js
import { handlers } from 'test/server-handlers';
const server = setupServer(...handlers);
```

I really like how kent's using aria roles in the tests to low-key ensure accessibility standards, ex:
```js
  expect(screen.getByRole('alert')).toBeInTheDocument();
```

## use inlinesnapshots for err msgs
`toMatchInlineSnapshot` is some wizard shit...look at this.

```js
expect(screen.getByRole('alert')).toMatchInlineSnapshot()
```
becomes (after running test):
```js
  expect(screen.getByRole('alert')).toMatchInlineSnapshot(`
    <div
      role="alert"
      style="color: red;"
    >
      password required
    </div>
  `)
```

jest will update our code for us. crazy. now we dont really want the div and associated styles bc again...thats implementation details. so we can just grab the text content:
```js
  expect(screen.getByRole('alert').textContent()).toMatchInlineSnapshot('password required')
```
and if we want to update the snapshot (which we will in this case, as first we included jsx and now its just text), we can type `u` in the terminal to update the jest snapshot. also this is helpful if say, the err msg changes.


## use one-off server handlers in your tests
we can add server handlers after the server has already started via `server.use()`. between tests, we can remove these handlers via `server.resetHandlers()` in order to preserve test isolation and restore the OG server handlers.
```js
test('unknown server error displays the error message', async () => {
  const testErrorMessage = 'Oh no, something bad happened'
  server.use(
    rest.post(
      'https://auth-provider.example.com/api/login',
      async (req, res, ctx) => {
        return res(ctx.status(500), ctx.json({message: testErrorMessage}))
      },
    ),
  )
  render(<Login />)
  userEvent.click(screen.getByRole('button', {name: /submit/i}))

  await waitForElementToBeRemoved(() => screen.getByLabelText(/loading/i))

  expect(screen.getByRole('alert')).toHaveTextContent(testErrorMessage)
})
```

^ this is beneficial in the case that we want to override the server behavior within a single test (ex: cause a 500 failure) to test that the results are as a user sees them.


## recap
so to recap...we use the following tools to mock http requests:

```js
import {rest} from 'msw'
import {setupServer} from 'msw/node'
```

this mock server allows us to **intercept endpoint requests and provide responses**:
```js
const server = setupServer(
  rest.get('/greeting', (req, res, ctx) => {
    return res(ctx.json({greeting: 'hello there'}))
  }),
)

beforeAll(() => server.listen())
afterEach(() => server.resetHandlers())
afterAll(() => server.close())
```

this way we setup and tear down the server after every test, and can now test endpoints which call the associated mocked endpoint.

we can also preserve these endpoints elsewhere for future reference AND use inline snapshots to prevent us having to manually add data/content to our jest assertions.

we can use one-off server handlers which allow us to override the mock server's behavior within a specific test, so we can test out what happens say, when a 500 err is hit for our component.

# mocking browser APIs and modules.

**Mocking HTTP requests is one thing, but sometimes you have entire Browser APIs or modules that you need to mock. Every time you create a fake version of what your code actually uses, you're "poking a hole in reality" and you lose some confidence as a result (which is why E2E tests are critical). Remember, we're doing it and recognizing that we're trading confidence for some practicality or convenience in our testing.**

for this course, when we are running tests with react-testing-library, we're actually running tests within a simulated node browser via JS-dom library. so it doesnt have everything a normal browser does. for those scenarios where it doesnt have something, we'll need to mock out that functionality.

Also, sometimes a module will do something that you dont want it to do for your tests.

we create mocks for things we typically cannot control .

to create mocks, we have to be as close to the API in our mocking as possible. This of course, requires knowing the API.

ex (getting latitude and longitude):
```js
beforeAll(() => {
  window.navigator.geolocation = {
    getCurrentPosition: jest.fn()
  };
})


test('displays the users current location', async () => {
  const fakePosition = {
    coords: {
      latitude: 35,
      longitude: 139
    }
  };

  const { promise, resolve, reject } = deferred()

  window.navigator.geolocation.getCurrentPosition.mockImplementation(callback => {
    promise.then(() => callback(fakePosition))
  });

  render(<Location />)
  expect(screen.getByLabelText(/loading/i)).toBeInTheDocument()

  await act(async () => {
    resolve();
    await promise;
  })

  expect(screen.queryByLabelText(/loading/i)).not.toBeInTheDocument()

  expect(screen.getByText(/latitude/i)).toHaveTextContent(`Latitude: ${fakePosition.coords.latitude}`);
  expect(screen.getByText(/longitude/i)).toHaveTextContent(`Longitude: ${fakePosition.coords.longitude}`);
})
```
^ here we say before the test, mock `getCurrentPosition` as a jest fn, which allows us to the use `mockImplementation()` to mock out the actual api (in our example, [getCurrentPosition()](https://developer.mozilla.org/en-US/docs/Web/API/Geolocation/getCurrentPosition) needs a success callback...failure callback optional)...so we need to mock that out. after we'e called that callback (through our self-created `deferred()` fn...dont like this), we can expect it to show up.

so:
- to mock out something, we set it as a `jest.fn()`, which then allows us to use `mockImplementation()` so we can mimic its API (hint hint: we should know the API first) to return out what we need for the test to be sucessful.

also lets talk about this utility fn:
```js
function deferred() {
  let resolve, reject
  const promise = new Promise((res, rej) => {
    resolve = res
    reject = rej
  })
  return { promise, resolve, reject }
}

const { promise, resolve, reject } = deferred()
```
^ this allows us to test for things in between states aka "first test loading, then resolve the promise manually, then test for the finish state, etc"

# testing custom hooks
typically dont, as its an implementation detail most of the time...maybe test the components that actually use them. so for that case, it might be useful to create a component that uses it and test that instead.

but, sometimes our components are mad complex and it would be a bunch of setup to test it out. we can create a stripped down version which just uses the custom hook:
```js
import { act } from 'react-dom/test-utils';

test('exposes the count and increment/decrement functions on test component', () => {
  let result
  function TestComponent(props) {
    result = useCounter(props)
    return null
  }

  render(<TestComponent />);

  expect(result.count).toBe(0)
  act(() => result.increment())
  expect(result.count).toBe(1)
  act(() => result.decrement())
  expect(result.count).toBe(0)
})
```
act() tries to update like a browser does, so it flushes the component effects and allows react to handle updating state, allowing for this custom hook's state to update and things to work without test errs

also peep `react-hooks` from react-testing-library for specifically testing hooks