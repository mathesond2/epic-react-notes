These are my course notes for Kent C Dodd's epicreact.dev course, easily one of the best courses I've taken and fully worth the $$. Each individual course has its own markdown file of notes...hope you find it useful 🏄‍♂️


# learnings summary
* understanding that this:
```js
React.useState(() => someExpensiveComputation())
```
is the same as this:
```js
React.useState(someExpensiveComputation);
```
* derived state vs. managed state - dont sync state, derive state constants from existing state to make state mgmt simpler.
* DO ASYNC CALLS WITHIN USEEFFECT AND NOT EVENT HANDLERS in order to be able to accurately take care of race conditions and cancellation events
* you cant run async fns directly in useEffect bc useEffect just returns a cleanup fn, you'll have to either define a fn and call it there, an IIFE, or define the fn outside the useEffect

* familiarity w react hooks like useMemo, useCallback, in addition to less familiar ones like useLayoutEffect and useDebug
* referential equality in objs/arrays and their reason in useCallback
* custom hooks as a way to encapsulate reused behavior (basically functions that use react hook)
* using context

* compound component pattern as a prime example of inversion of control in order to make components more flexible
* state reducer pattern and prop getter pattern to allow inversion of control re: letting users put their own stuff into our custom hook
* prop collection pattern in order to bake often-used props for our hooks (to ensure expected behavior, think accessibility)

* the reason for memoization via useMemo and useCallback
* creating custom hooks as a proxy for context in order to account for err handling, side effects, etc so users interact w the hook instead of the context directly.
* bundle custom hooks and associated functions within an exported context Provider to further encapsuilate related functionality and be used in conjunction with the custom hook for access to the context
* bundling async calls and their associated states (isLoading, isError, data, etc) within a custom hook

* `React.cloneElement` in order to add functionality to children components

* using react testing library
* using testingplayground.com for react-testing-library helpers
* your tests should mostly reflect how your users use your app
* dont test implementation details (ex: wanna test a custom hook? test where its used from a user pov...what does it DO) in order to make your tests more resilient to change and to ensure expected app behavior for USERS
* mocking fns in jest, mocking http requests, what mocking actually does....mocked fns are known as 'spies' bc we can indirectly test that fns are called rather than just testing the output of something. check out this component.
* using area roles in tests to low-key ensure accessiblity standards
* tools like msw in order to mock http requests by creating a mock server that intercepts endpoint reqs and provides responses
* mocking browser APIs: " Every time you create a fake version of what your code actually uses, you're "poking a hole in reality" and you lose some confidence as a result (which is why E2E tests are critical). Remember, we're doing it and recognizing that we're trading confidence for some practicality or convenience in our testing."
  * to create a mock, you need to be as close to the API as possible, AND likewise you have to know the API as well.



# a sample of how you can use all this today:
* react testing library
* mocking browser APIs in tests
* mocking http reqs
* using testingplayground.com for react-testing-library helpers
* how/why to not test implementation details

* authentication via context custom hook and custom provider as well to abstract out stuff
* custom hook for api endpoint requests? eh, might be a bit much
* creating compound components
* more creation of custom hooks to encapsulate functionality
* more informed usage of react hooks like useState, useEffect, useCallback, useMemo, etc
* more informed usage of using context
* derived state vs. managed state to avoid syncing state in general
* custom hooks for loading/error/loadedonce states.
* making your global state as lean as possible
* **inversion of control** in code in order to have a more clear API and allow users of your code (components, hooks, fns, etc) to insert their use cases into it.
* the reason for hook dep arrays
* lazy state initialization for expesnsive renders
* `blah={() => something()}` is same as `blah={something}`
* useReducer for multiple elements of state that typically change together
* memoization - cache a value so you dont have to calculate it everytime
* referential equality in general
* use component composition to flatten app component tree
* use context to avoid prop drilling
* useCallback's usage mostly for fns that are part of other hooks' dep arrays to avoid re-renders
* using error boundaries