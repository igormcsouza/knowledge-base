---
tags:

- web-development
- react
- javascript
- typescript
- frontend

---

# React Principles

React becomes easier to reason about when the mental model is clear: state changes cause renders, props flow through component boundaries, Effects synchronize with external systems, and different kinds of state should be managed differently.

## Rendering and State

A simplified React cycle is:

\`\`\`
User interaction
      |
      v
setState / dispatch
      |
      v
React renders
      |
      v
React commits UI changes
      |
      v
Effects run when their dependencies require them
\`\`\`

State should be treated as immutable data:

\`\`\`javascript
// Bad
user.name = name;
setUser(user);

// Good
setUser({
  ...user,
  name,
});
\`\`\`

When updating state based on previous state, use the functional form:

\`\`\`javascript
setCount(current => current + 1);
setCount(current => current + 1);
setCount(current => current + 1);
\`\`\`

Using setCount(count + 1) three times can reuse the same value from the current render.

## Derived Data

Do not create state for values that can be calculated from existing state.

Prefer:

\`\`\`javascript
const filteredProducts = products.filter(
  product => product.name.includes(search)
);
\`\`\`

over maintaining filteredProducts as separate state and synchronizing it with an Effect.

This avoids duplicated state and unnecessary Effects.

## useEffect

useEffect is primarily for synchronizing React with an external system such as APIs, browser APIs, timers, WebSockets, subscriptions, and third-party libraries.

\`\`\`javascript
useEffect(fn);       // after every render
useEffect(fn, []);   // after the initial mount
useEffect(fn, [x]);  // after mount and when x changes
\`\`\`

The dependency array describes reactive values the Effect depends on.

For an API request that should happen after the component mounts:

\`\`\`javascript
useEffect(() => {
  getProducts();
}, []);
\`\`\`

External resources should be cleaned up:

\`\`\`javascript
useEffect(() => {
  const socket = new WebSocket("wss://example.com/events");

  socket.onmessage = event => {
    // update state
  };

  return () => {
    socket.close();
  };
}, []);
\`\`\`

## Race Conditions in Effects

Search requests can race:

\`\`\`
"j"     -> request A
"jo"    -> request B
"joh"   -> request C
"john"  -> request D
\`\`\`

Responses are not guaranteed to finish in that order. An older request can overwrite newer data.

Use cancellation when the previous request is obsolete:

\`\`\`javascript
useEffect(() => {
  const controller = new AbortController();

  fetch(\`/api/users?q=\${query}\`, {
    signal: controller.signal,
  })
    .then(response => response.json())
    .then(setUsers)
    .catch(error => {
      if (error.name !== "AbortError") {
        console.error(error);
      }
    });

  return () => controller.abort();
}, [query]);
\`\`\`

For search inputs, debouncing can also prevent unnecessary requests.

## useRef

useRef is a persistent mutable container whose changes do not trigger a render.

The simplest use is a DOM reference:

\`\`\`jsx
const inputRef = useRef(null);

return (
  <>
    <input ref={inputRef} />
    <button onClick={() => inputRef.current?.focus()}>
      Focus
    </button>
  </>
);
\`\`\`

A useful rule:

- useState: remember a value and update the UI when it changes.
- useRef: remember a value without causing a render when it changes.

Refs are also useful for timers, previous values, AbortControllers, and other imperative values.

## useReducer

When several state transitions belong together, useReducer can make the state model explicit:

\`\`\`javascript
const [state, dispatch] = useReducer(reducer, initialState);

dispatch({ type: "SAVE_START" });
dispatch({ type: "SAVE_SUCCESS", payload: product });
\`\`\`

The model becomes:

\`\`\`
UI event
  |
  v
dispatch(action)
  |
  v
reducer
  |
  v
new state
  |
  v
render
\`\`\`

This is also useful preparation for Redux.

## Context and Prop Drilling

Prop drilling happens when data is passed through components that do not use it only so a deeper component can receive it.

Context can provide values to descendants without explicitly passing them through every intermediate component.

Context is not a universal state-management solution. It is primarily a mechanism for making values available to a component subtree.

## Client State vs. Server State

Client state includes modal visibility, selected product, sidebar state, theme, local form state, and cart state.

Server state includes products, orders, customers, and dashboard statistics from an API.

Server state has additional concerns:

- caching
- stale data
- refetching
- retries
- deduplication
- synchronization

## React Query

A custom hook can manually implement fetching, loading, error, data, caching, retries, refetching, and deduplication.

TanStack Query provides an abstraction around server state:

\`\`\`javascript
const {
  data,
  isLoading,
  error,
} = useQuery({
  queryKey: ["products", search],
  queryFn: () => getProducts(search),
});
\`\`\`

The important concept is understanding why server state is different from local UI state.

## RTK Query

RTK Query solves a similar server-state/data-fetching problem within the Redux Toolkit ecosystem.

\`\`\`
React Query / TanStack Query
    |
    +-- server-state/data-fetching abstraction

RTK Query
    |
    +-- server-state/data-fetching integrated with Redux Toolkit
\`\`\`

Choose based on the application's state architecture rather than assuming all state belongs in Redux.

## useMemo, useCallback and React.memo

useMemo memoizes a calculated value:

\`\`\`javascript
const filteredProducts = useMemo(() => {
  return products
    .filter(...)
    .sort(...);
}, [products, search, sort]);
\`\`\`

useCallback memoizes a function reference:

\`\`\`javascript
const handleSelect = useCallback((product) => {
  setSelectedProduct(product);
}, []);
\`\`\`

React.memo allows a component to skip rendering when its props are equal according to its comparison.

\`\`\`
useMemo     -> memoized value
useCallback -> memoized function
React.memo  -> memoized component rendering
\`\`\`

Do not use memoization everywhere. It is a performance optimization, not a correctness requirement. A good reason for useCallback is maintaining a stable callback passed to a memoized child. A good reason for useMemo is avoiding an expensive calculation when its inputs have not changed.

## Keys

Keys allow React to identify list items across renders:

\`\`\`jsx
users.map(user => (
  <User key={user.id} user={user} />
))
\`\`\`

Stable IDs are preferable to array indexes when items can be inserted, removed, or reordered.

## Component Architecture

A large component containing API calls, form state, validation, tables, pagination, modals, business rules, and formatting has too many responsibilities.

A useful decomposition is:

\`\`\`
Page
 |
 +-- ProductForm
 +-- ProductTable
 +-- Pagination
 +-- ProductModal

Data / behavior
 |
 +-- useProducts()
 +-- usePagination()
 +-- useProductForm()

Pure logic
 |
 +-- formatPrice()
 +-- validateProduct()
\`\`\`

The goal is not to split code simply because it can be split. The goal is to separate responsibilities and make boundaries meaningful.

## SSR, SSG and CSR

CSR means Client-Side Rendering: the browser loads the JavaScript application and React renders the page on the client.

SSR means Server-Side Rendering: the server generates HTML for a request and sends it to the browser.

SSG means Static Site Generation: HTML is generated ahead of time, usually during a build, and served as static content.

A simple mental model:

\`\`\`
CSR -> browser generates
SSR -> server generates per request
SSG -> generated ahead of time
\`\`\`

Hydration is the process of React attaching its behavior to server-rendered HTML so the page becomes interactive.

SSR is a rendering strategy; it is not automatically a security mechanism.

## WebSockets

WebSockets are useful when communication needs to be persistent and bidirectional:

\`\`\`
HTTP
Client ---- request ----> Server
Client <--- response ---- Server

WebSocket
Client <=================> Server
          persistent
          full-duplex
\`\`\`

In React, a WebSocket is an external system, so its lifecycle fits naturally into an Effect:

\`\`\`javascript
useEffect(() => {
  const socket = new WebSocket("wss://example.com/events");

  socket.onmessage = event => {
    const data = JSON.parse(event.data);
    setNotifications(current => [...current, data]);
  };

  return () => socket.close();
}, []);
\`\`\`

For server-only push, SSE can be simpler. For infrequent updates, polling may be sufficient.

See [WebSockets](websockets.md) for the protocol and scaling details.

## Accessibility

Accessibility should be part of component design, not a final check.

Prefer semantic HTML:

Use a semantic button element, for example: `button` with the accessible name `Save`.

over a clickable generic element.

Important areas include semantic HTML, labels, keyboard navigation, focus management, color contrast, accessible names, alternative text, appropriate ARIA usage, and screen-reader behavior.

WCAG organizes accessibility around four principles:

- Perceivable
- Operable
- Understandable
- Robust

Google Lighthouse and tools such as axe can audit an application.

Reference: [W3C Web Accessibility Initiative](https://www.w3.org/WAI/standards-guidelines/wcag/)

## Tailwind CSS

Tailwind provides utility classes that can be composed directly in markup:

For example, a button can compose utilities such as `px-4`, `py-2`, `rounded-md`, and `font-medium`.

Know responsive utilities, spacing, flexbox/grid, hover/focus states, breakpoints, dark mode, and utility composition.

Tailwind's JIT approach generates CSS from the classes used by the project instead of requiring a large pre-generated utility stylesheet.

The qualification goal is practical fluency rather than memorizing every class.

## Performance

Performance optimization should start with measurement.

Useful tools include React DevTools Profiler, Chrome DevTools Performance and Memory, and Lighthouse.

Ask:

1. Which component rendered?
1. Why did it render?
1. How long did it take?
1. Is the work actually expensive?
1. Can the problem be solved structurally before adding memoization?

For very large lists, virtualization renders only the visible portion instead of creating DOM nodes for every item.

## Qualification Mental Model

\`\`\`
State
  |
  +-- local UI state -> useState
  +-- complex transitions -> useReducer
  +-- shared client state -> Context / Redux
  +-- server state -> React Query / RTK Query

External systems
  |
  +-- API -> Effect / data-fetching library
  +-- WebSocket -> Effect + cleanup
  +-- DOM -> useRef

Performance
  |
  +-- measure first
  +-- React.memo
  +-- useMemo
  +-- useCallback
  +-- virtualization
\`\`\`

The strongest answers explain the reason behind each tool, what problem it solves, and what complexity it introduces.

## Related Articles

- [JavaScript Fundamentals: Event Loop, Promises & Closures](javascript-fundamentals.md)
- [TypeScript Fundamentals](typescript-fundamentals.md)
- [WebSockets](websockets.md)
- [FastAPI Event Loop](fastapi-event-loop.md)
