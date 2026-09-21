---
tags:

- web-development
- javascript
- async
- event-loop
- closures

---

# JavaScript Fundamentals: Event Loop, Promises & Closures

JavaScript is single-threaded at the level of a JavaScript execution context: one Call Stack executes JavaScript at a time. This is not the same thing as Python's CPython GIL. The runtime can provide multiple execution contexts, such as Web Workers, while the main JavaScript context still has one Call Stack.

The browser or Node.js runtime provides timers, networking, DOM events, and the Event Loop around the JavaScript engine.

## The Call Stack

Synchronous JavaScript is executed on the Call Stack:

\`\`\`javascript
console.log("A");
console.log("B");
console.log("C");
\`\`\`

A long-running synchronous operation blocks the stack. The runtime does not automatically move arbitrary CPU-heavy JavaScript elsewhere.

## The Event Loop

The Event Loop coordinates when asynchronous work can return to the JavaScript Call Stack.

\`\`\`
JavaScript
    |
    v
Call Stack
    |
    +----> Runtime APIs
    |       +--> timers
    |       +--> fetch/networking
    |       +--> DOM events
    |
    v
Queues
    |
    v
Event Loop
    |
    v
Call Stack
\`\`\`

Async and await are not required for the Event Loop to work.

\`\`\`javascript
console.log("A");

setTimeout(() => {
  console.log("B");
}, 0);

console.log("C");
\`\`\`

The result is:

\`\`\`
A
C
B
\`\`\`

A zero-delay timer does not mean "execute immediately". The callback still waits until it is eligible and the JavaScript stack is available.

## Tasks and Microtasks

In simplified terms:

- Tasks (often called macrotasks) include timer callbacks and many event callbacks.
- Microtasks include Promise continuations such as then, catch, finally, and queueMicrotask.
- After the current synchronous work finishes, pending microtasks are processed before the runtime proceeds to the next task.

\`\`\`javascript
console.log("A");

setTimeout(() => {
  console.log("B");
}, 0);

Promise.resolve().then(() => {
  console.log("C");
});

console.log("D");
\`\`\`

The result is:

\`\`\`
A
D
C
B
\`\`\`

## Promises and async/await

A Promise represents the eventual result of an asynchronous operation.

\`\`\`javascript
fetch("/api/users")
  .then(response => response.json())
  .then(users => {
    console.log(users);
  });
\`\`\`

Async/await is syntax for working with Promise-based asynchronous code:

\`\`\`javascript
async function loadUsers() {
  const response = await fetch("/api/users");
  const users = await response.json();

  return users;
}
\`\`\`

Await does not block the entire JavaScript runtime. It suspends the current async function's continuation while the Promise is pending.

## Closures

A closure happens when a function retains access to variables from the lexical scope where it was created, even after the outer function has returned.

\`\`\`javascript
function createCounter() {
  let count = 0;

  return function () {
    count += 1;
    return count;
  };
}

const counter = createCounter();

console.log(counter()); // 1
console.log(counter()); // 2
console.log(counter()); // 3
\`\`\`

Closures are useful for private state, function factories, callbacks, event handlers, and maintaining state across calls.

They are also fundamental to React. A function created during a render closes over the values from that render:

\`\`\`jsx
function Counter() {
  const [count, setCount] = useState(0);

  function handleClick() {
    console.log(count);
  }

  // handleClick closes over this render's count.
}
\`\`\`

## Event Loop vs. Python asyncio

The mental model is similar to FastAPI's asyncio model:

| Python / FastAPI | JavaScript |
|---|---|
| asyncio event loop | runtime Event Loop |
| coroutine | async function |
| await | await |
| async I/O | runtime APIs such as fetch |
| task scheduling | callback/Promise scheduling |

The analogy should not be taken literally. CPython's GIL is a locking mechanism related to Python bytecode execution; the JavaScript Event Loop is a scheduling mechanism.

## Practical Rule

When analyzing asynchronous JavaScript, ask:

1. What is executing synchronously on the Call Stack?
1. What operation is being handled by the runtime?
1. Does it produce a Promise or callback?
1. Which queue will the continuation enter?
1. When will the Event Loop allow it to execute?

## Related Articles

- [TypeScript Fundamentals](typescript-fundamentals.md)
- [FastAPI Event Loop](fastapi-event-loop.md)
- [WebSockets](websockets.md)
- [React Principles](react-principles.md)
