---
tags:

- web-development
- typescript
- javascript
- types

---

# TypeScript Fundamentals: Typing Properly

TypeScript adds a static type system on top of JavaScript, checked at compile
time (`tsc`) rather than at runtime. The goal isn't "add types everywhere" —
it's to let the compiler catch entire classes of bugs (wrong shape passed to
a function, typo'd property, `undefined` slipping through) before the code
ever runs.

## Basic Types

```typescript
let name: string = "Igor";
let age: number = 30;
let active: boolean = true;
let tags: string[] = ["python", "typescript"];
let point: [number, number] = [10, 20]; // tuple: fixed length, fixed types

enum Role {
  Admin,
  Editor,
  Viewer,
}
```

Most of the time, explicit annotations on simple variables aren't needed —
TypeScript's **type inference** figures it out from the assigned value:

```typescript
let name = "Igor"; // inferred as string, no annotation needed
```

Reach for explicit annotations where inference can't help: function
parameters, function return types on public APIs, and empty structures
(`let items: string[] = []`, since `[]` alone infers as `any[]`).

## `any`, `unknown`, `never`, `void`

- **`any`** — opts a value out of type checking entirely. It's the escape
  hatch; every operation on an `any` is allowed and unchecked. Avoid it —
  it silently defeats the point of using TypeScript at all.
- **`unknown`** — also "could be anything," but safe: you can't do anything
  with an `unknown` value until you narrow it to a specific type first. Use
  `unknown` for things like API responses or `catch` clause errors, instead
  of `any`.
- **`never`** — a function that never returns (always throws, or loops
  forever) returns `never`. Also shows up as the type of an unreachable
  branch, which is useful for exhaustiveness checks.
- **`void`** — a function that doesn't return a meaningful value.

```typescript
function parseInput(data: unknown): string {
  if (typeof data !== "string") {
    throw new Error("expected a string"); // return type of this branch: never
  }
  return data; // narrowed to string here
}
```

## Interfaces vs. Type Aliases

Both describe object shapes; the choice mostly comes down to conventions and
a couple of real differences:

```typescript
interface User {
  id: number;
  name: string;
  email?: string; // optional property
  readonly createdAt: Date; // can't be reassigned after creation
}

type UserId = number | string; // type aliases can name unions, interfaces can't
```

- `interface` supports **declaration merging** (two `interface User {}`
  declarations in scope combine into one) and reads naturally with `extends`
  for object inheritance — a common convention for public object/class
  shapes.
- `type` can alias anything — unions, tuples, primitives, mapped types — not
  just object shapes, so it's the only option for things like
  `type Status = "pending" | "done" | "failed"`.

A reasonable default: `interface` for object shapes you expect to be
extended or implemented; `type` for unions, tuples, and anything that isn't
a plain object shape.

## Function Typing

```typescript
function add(a: number, b: number): number {
  return a + b;
}

function greet(name: string, greeting: string = "Hello"): string {
  return `${greeting}, ${name}`;
}

function log(message: string, code?: number): void {
  console.log(code ? `[${code}] ${message}` : message);
}
```

Parameters and return types get annotated explicitly; TypeScript won't infer
a function's parameter types from how it's called elsewhere.

## Union & Intersection Types

```typescript
type Id = number | string; // union: could be either
type Employee = Person & { salary: number }; // intersection: must satisfy both
```

## Type Narrowing

TypeScript tracks control flow and narrows a variable's type inside
conditional branches:

```typescript
function formatId(id: number | string): string {
  if (typeof id === "number") {
    return id.toFixed(0); // id is `number` here
  }
  return id.toUpperCase(); // id is `string` here
}
```

**Discriminated unions** — a shared literal field used to narrow between
variant shapes — are one of the most useful patterns in TypeScript:

```typescript
type Shape =
  | { kind: "circle"; radius: number }
  | { kind: "rectangle"; width: number; height: number };

function area(shape: Shape): number {
  switch (shape.kind) {
    case "circle":
      return Math.PI * shape.radius ** 2; // narrowed to the circle variant
    case "rectangle":
      return shape.width * shape.height; // narrowed to the rectangle variant
  }
}
```

## Generics

Generics let a function, interface, or class stay type-safe while working
over more than one concrete type:

```typescript
function first<T>(items: T[]): T | undefined {
  return items[0];
}

first([1, 2, 3]); // inferred as number | undefined
first(["a", "b"]); // inferred as string | undefined

interface Box<T> {
  value: T;
}

const numberBox: Box<number> = { value: 42 };
```

Without the generic, `first` would either lose type information (`any`) or
need one copy per type. `<T>` keeps the relationship between input and output
types intact.

## Useful Utility Types

Built into TypeScript, these transform existing types instead of writing new
ones from scratch:

```typescript
interface User {
  id: number;
  name: string;
  email: string;
}

type PartialUser = Partial<User>;   // all properties optional
type UserPreview = Pick<User, "id" | "name">; // only id + name
type UserWithoutId = Omit<User, "id">;        // everything except id
type UserMap = Record<number, User>;          // { [key: number]: User }
type ReadonlyUser = Readonly<User>;           // all properties readonly
```

## Best Practices

- Enable `strict` mode in `tsconfig.json` (`"strict": true`) — it turns on
  `strictNullChecks`, `noImplicitAny`, and friends, which catch the majority
  of real bugs TypeScript is good at catching.
- Prefer `unknown` over `any` at boundaries (API responses, `JSON.parse`
  results) and narrow before use.
- Let inference do the work for local variables; annotate function
  signatures explicitly since they're the contract other code relies on.
- Model states with discriminated unions instead of optional fields plus
  booleans (`{ status: "loading" } | { status: "success", data: T }` beats
  `{ loading: boolean, data?: T }`).

## Related Articles

- [FastAPI Event Loop](fastapi-event-loop.md) — the backend-side equivalent
  concern of knowing exactly what runs where.
