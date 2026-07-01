# Foundations (1-3)
### JavaScript Core, TypeScript, Async & Event-Driven Programming

---

# 1. JavaScript Core

## Why JavaScript Fundamentals Still Matter for Senior Roles

Strong foundations separate senior engineers from mid-level ones. Interviewers probe closures, `this`, the prototype chain, and execution model specifically because frameworks abstract away the details — but bugs live in the abstractions. You can build React apps for years without truly understanding closures, and then get blindsided by a stale variable in a callback. This section covers the mechanisms that underpin everything else in the language.

---

## 1.1 Closures & Scope

### What are Scope and Closures?

**Scope** defines where a variable is accessible. JavaScript has three kinds:
- **Global scope** — accessible everywhere in the program
- **Function scope** — variables declared with `var` exist only within the function they were declared in, regardless of any nested blocks
- **Block scope** — variables declared with `let` or `const` exist only within the `{}` block they were declared in (e.g. inside an `if`, `for`, etc.)

**Closure** is what happens when a function "remembers" the variables from the scope where it was defined, even after that scope has finished executing. Every function in JavaScript forms a closure — it carries a reference to its outer scope with it wherever it goes.

A good interview answer: *"Scope determines which variables are visible where. A closure is a function that retains access to variables from its outer scope even after that scope has returned. This lets us do things like data encapsulation, memoization, and factory functions — because the inner function keeps a live reference to the outer variables, not a copy."*

### Lexical Scope

Scope is determined at write time by where a function is defined, not where it is called. An inner function always has access to the variables of its enclosing scope, even after the outer function has returned.

```javascript
function outer() {
  const x = 10;
  function inner() {
    console.log(x); // 10 — x is visible here because inner is defined inside outer
  }
  return inner;
}
const fn = outer();
fn(); // 10 — outer() has already returned, but x persists in fn's closure
```

A **closure** is a function bundled together with references to its surrounding lexical environment. The function retains access to variables in its outer scope even after the outer function returns.

**"Lexical"** means scope is determined by **where the code is written**, not when or how it runs. The engine sees `inner` nested inside `outer` at parse time and wires up the scope chain before any code executes. So `inner` can access `x` because it was *defined* inside `outer` — even when called later after `outer` has already returned and `x` would otherwise be gone.

### Classic Closure Pitfall (`var` in loops)

This is one of the most common JavaScript interview questions. Understanding it requires knowing both `var`'s scoping rules and how the event loop defers callbacks.

```javascript
// Bug: all closures share the same `i` (function-scoped)
for (var i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 0); // 3, 3, 3
}
// Why 3, 3, 3? Two things combine to cause this:
// 1. var is function-scoped, not block-scoped — there is only ONE `i`
//    variable shared across all iterations, not a fresh one per iteration.
// 2. setTimeout with delay 0 doesn't run immediately — it queues the
//    callback in the macrotask queue. The entire for loop finishes first
//    (synchronous code always runs before queued callbacks), so by the
//    time any callback executes, `i` has already been incremented to 3.

// Fix 1: let (block-scoped — each iteration gets its own fresh binding)
for (let i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 0); // 0, 1, 2
}
// `let` creates a new `i` binding per iteration of the loop, so each
// arrow function closes over a different variable.

// Fix 2: IIFE captures the current value by passing it as an argument
for (var i = 0; i < 3; i++) {
  ((j) => setTimeout(() => console.log(j), 0))(i); // 0, 1, 2
}
// The IIFE is called immediately with the current value of i,
// creating a new scope with its own j. This was the pre-let workaround.
```

### Practical Closure Uses

Closures aren't just an academic concept — they're the mechanism behind some of the most common patterns in JavaScript.

```javascript
// ── Data encapsulation ─────────────────────────────────────────────
// Without closures, `count` would need to be a global or a class field.
// Closures let us keep it truly private — nothing outside can read or
// modify it except through the returned methods.
function createCounter(start = 0) {
  let count = start; // private — inaccessible from outside
  return {
    increment: () => ++count,
    decrement: () => --count,
    get: () => count,
  };
}
const counter = createCounter(10);
counter.increment(); // 11
counter.increment(); // 12
counter.get();       // 12
// There is no way to set `count` to an arbitrary value from outside.

// ── Memoization ────────────────────────────────────────────────────
// The `cache` Map lives in the closure of the returned function.
// Every call to the memoized function shares the same cache —
// it persists as long as the memoized function exists.
function memoize(fn) {
  const cache = new Map(); // closed over — persists between calls
  return function (...args) {
    const key = JSON.stringify(args);
    if (cache.has(key)) return cache.get(key); // return cached result
    const result = fn.apply(this, args);        // compute once
    cache.set(key, result);                     // store for next time
    return result;
  };
}

const expensiveCalc = memoize((n) => {
  console.log('computing...');
  return n * n;
});
expensiveCalc(5); // logs "computing...", returns 25
expensiveCalc(5); // returns 25 immediately, no log
```

---

## 1.2 `this` Binding

### What is `this` and Why is it Confusing?

In most languages, a method knows which object it belongs to — it's determined by where the method is defined. JavaScript is different: `this` is **dynamic**, meaning its value is determined by *how* the function is called at runtime, not where it was written. This makes it one of the most common sources of bugs in JavaScript, especially when passing methods as callbacks.

There are four rules that determine what `this` refers to. They apply in this priority order — if multiple rules could apply, the higher one wins.

```javascript
// ── Rule 1: new binding ────────────────────────────────────────────
// When a function is called with `new`, a fresh object is created,
// `this` is set to that object, and it is returned automatically.
function Person(name) {
  this.name = name; // `this` is the new object being constructed
}
const alice = new Person('Alice');
alice.name; // "Alice"

// ── Rule 2: Explicit binding ────────────────────────────────────────
// call() and apply() let you specify what `this` should be.
// bind() returns a new function with `this` permanently fixed.
function greet(greeting) {
  return `${greeting}, ${this.name}`;
}
greet.call({ name: 'Bob' }, 'Hello');        // "Hello, Bob"
greet.apply({ name: 'Bob' }, ['Hello']);     // "Hello, Bob" — args as array
const boundGreet = greet.bind({ name: 'Carol' });
boundGreet('Hi');                            // "Hi, Carol" — `this` is fixed forever

// ── Rule 3: Implicit binding ────────────────────────────────────────
// When a function is called as a method of an object, `this` is the
// object to the left of the dot at the call site.
const obj = {
  name: 'Dave',
  greet() { return this.name; },
};
obj.greet(); // "Dave" — `this` = obj

// Watch out: extracting the method loses the binding!
const fn = obj.greet;
fn(); // undefined (strict) — no longer called as obj.greet

// ── Rule 4: Default binding ─────────────────────────────────────────
// A plain function call with no object, `new`, or explicit binding.
// In strict mode: `this` is undefined. In sloppy mode: globalThis.
function show() { console.log(this); }
show(); // undefined (strict mode) / window or global (sloppy)
```

### Arrow Functions — Lexical `this`

Arrow functions do not have their own `this`. Instead, they capture `this` from the enclosing scope **at the time they are defined** — this is called lexical `this`. This makes them ideal for callbacks inside methods, where you want to refer to the surrounding object.

```javascript
class Timer {
  constructor() {
    this.ticks = 0;
  }

  start() {
    // Arrow function: `this` is captured from `start()`'s scope,
    // which is the Timer instance. Works correctly.
    setInterval(() => {
      this.ticks++; // `this` = Timer instance ✅
    }, 1000);

    // Regular function: `this` is determined by how setInterval calls it,
    // which is a plain call — so `this` is undefined in strict mode.
    // setInterval(function() { this.ticks++; }, 1000); // TypeError ❌
  }
}
```

**Rule of thumb:** Use arrow functions for callbacks inside methods. Use regular functions when you want callers to control `this` (e.g. event handlers, constructor functions, prototype methods). Arrow functions cannot be used with `new` and `bind`/`call`/`apply` will not change their `this`.

---

## 1.3 Prototypes & Class Sugar

### What is the Prototype Chain?

JavaScript's inheritance model is fundamentally different from class-based languages like Java or C#. Instead of classes creating instances, JavaScript uses **prototype delegation**: objects link to other objects, and when you access a property that doesn't exist on an object, JavaScript walks up the chain of linked objects until it either finds it or reaches `null`.

Every object has an internal `[[Prototype]]` link (accessible via `Object.getPrototypeOf()`). When you do `obj.method()`, JavaScript checks `obj` first, then `obj`'s prototype, then its prototype's prototype, and so on. This chain is what makes inheritance work.

Understanding this matters because:
- The `class` keyword introduced in ES2015 is syntactic sugar — it still uses prototypes underneath
- Bugs with prototype mutation can affect every instance of a "class"
- Interviewers frequently ask you to explain what `class` actually compiles to

### Prototype-Based Inheritance (Pre-Class)

Before `class` syntax, inheritance was wired manually through constructor functions and the prototype chain. Understanding this is essential because `class` is syntactic sugar over the same mechanism — knowing the desugared form lets you reason about `instanceof`, prototype pollution, and why shared methods don't belong on `this`.

```javascript
// Constructor function — called with `new`
function Animal(name) {
  this.name = name; // instance property, stored on each object
}
// Methods go on the prototype — shared across all instances
// (not copied onto each object, saving memory)
Animal.prototype.speak = function () {
  return `${this.name} speaks`;
};

function Dog(name) {
  Animal.call(this, name); // call parent constructor to set this.name
}
// Set up the prototype chain: Dog.prototype → Animal.prototype
Dog.prototype = Object.create(Animal.prototype);
// Restore the constructor reference (Object.create overwrites it)
Dog.prototype.constructor = Dog;
Dog.prototype.bark = function () {
  return `${this.name} barks`;
};

const d = new Dog('Rex');
d.speak(); // "Rex speaks" — not on d or Dog.prototype, found on Animal.prototype
d.bark();  // "Rex barks" — found on Dog.prototype
// Full chain: d → Dog.prototype → Animal.prototype → Object.prototype → null
```

### Class Syntax (ES2015+ sugar)

The `class` keyword makes inheritance readable and familiar, but it's important to know it doesn't introduce a new object model. It's still prototype delegation underneath.

```javascript
class Animal {
  #name; // private field — truly inaccessible outside the class (ES2022)
         // unlike convention-based _name, this is enforced by the engine

  constructor(name) {
    this.#name = name;
  }

  speak() {
    return `${this.#name} speaks`;
    // this method lives on Animal.prototype, not on each instance
  }

  static create(name) {
    // static methods live on the class itself, not on instances
    return new Animal(name);
  }
}

class Dog extends Animal {
  bark() {
    return 'Woof!';
    // lives on Dog.prototype
  }
  // `extends` sets up Dog.prototype → Animal.prototype automatically
  // `super()` in the constructor calls Animal's constructor
}

// Proof that `class` is just sugar:
typeof Animal;              // "function" — classes are functions
Animal.prototype.speak;     // the method is on the prototype, not instances
Object.getPrototypeOf(Dog.prototype) === Animal.prototype; // true
```

### `Object.create` & Pure Prototype Delegation

`Object.create(proto)` creates an object whose prototype is exactly `proto` — no constructor function, no `new`, no implicit properties. This is the purest expression of prototypal inheritance and the basis of patterns like the OLOO (Objects Linking to Other Objects) style.

```javascript
// Object.create(proto) creates a new object with `proto` as its prototype.
// No constructor function needed.
const animal = {
  speak() { return `${this.name} speaks`; },
};

const dog = Object.create(animal); // dog → animal → Object.prototype → null
dog.name = 'Rex';
dog.speak(); // "Rex speaks" — `speak` found via prototype delegation
```

---

## 1.4 Generators & Iterators

### What are Iterators?

An **iterator** is any object that knows how to produce a sequence of values one at a time. JavaScript formalises this with the **iterator protocol**: an object is an iterator if it has a `next()` method that returns `{ value, done }`. An object is **iterable** if it has a `[Symbol.iterator]()` method that returns an iterator.

This protocol is what powers `for...of`, spread (`...`), destructuring, `Array.from()`, and `Promise.all()` — they all work with any iterable, not just arrays.

The value of this abstraction is **lazy evaluation**: you can represent potentially infinite sequences (like a Fibonacci series or a stream of API pages) without computing all values upfront.

### Custom Iterator (Iterator Protocol)

Any object can be made iterable by implementing `[Symbol.iterator]`, which returns an iterator object with a `next()` method. Once iterable, the object works with `for...of`, spread (`...`), destructuring, and `Array.from`.

```javascript
// Making an object iterable by implementing [Symbol.iterator]
const range = {
  from: 1,
  to: 5,
  [Symbol.iterator]() {
    let current = this.from;
    const last = this.to;
    // Returns the iterator object with a next() method
    return {
      next() {
        if (current <= last) {
          return { value: current++, done: false }; // yield next value
        }
        return { value: undefined, done: true }; // sequence is exhausted
      },
    };
  },
};

for (const n of range) console.log(n); // 1, 2, 3, 4, 5
const arr = [...range]; // [1, 2, 3, 4, 5] — spread works because it's iterable
```

### Generator Functions

Generators are a cleaner way to create iterators. A generator function uses `function*` syntax and `yield` to pause execution and produce a value. The caller pulls values one at a time with `.next()` — the function body only runs up to the next `yield`, then suspends.

This is powerful because the function retains its local state between calls. You don't need to manually track `current` in a closure — the generator's execution context is automatically paused and resumed.

```javascript
// `function*` declares a generator. Calling it returns a generator object
// (which is both an iterator and an iterable), but runs NO code yet.
function* fibonacci() {
  let [a, b] = [0, 1];
  while (true) {
    yield a;             // pause here, return `a` as the next value
    [a, b] = [b, a + b]; // resumes here on the next .next() call
  }
}

const fib = fibonacci(); // no code runs yet
fib.next().value; // 0 — runs until first yield
fib.next().value; // 1
fib.next().value; // 1
fib.next().value; // 2
// Infinite sequence — only computes as much as you ask for

// ── Async generators — real-world use: paginated API ──────────────
// Combines async/await with generator semantics.
// The caller iterates through pages without knowing how many there are.
async function* paginate(url) {
  let page = 1;
  while (true) {
    const data = await fetch(`${url}?page=${page}`).then(r => r.json());
    if (!data.length) return; // done — signal end of sequence
    yield data;               // pause, give caller this page's data
    page++;                   // resumes here when caller asks for next page
  }
}

// Caller just iterates — no manual page tracking, no loading all pages upfront
for await (const batch of paginate('/api/users')) {
  processBatch(batch);
}
```

---

## 1.5 Proxy & Reflect

### What is Proxy?

`Proxy` lets you intercept and redefine fundamental operations on objects: property reads, writes, deletions, function calls, and more. You wrap an existing object (the *target*) with a `Proxy` and provide a *handler* object whose methods (called *traps*) are called instead of the default behaviour.

This is **metaprogramming** — writing code that controls how the language itself behaves for a given object. Real-world uses include:
- **Validation** — reject invalid property assignments
- **Reactivity** — trigger UI updates when data changes (Vue 3's entire reactivity system is built on Proxy)
- **Logging / observability** — transparently log all property access
- **Virtual objects** — fake objects that lazy-load their data on access

`Reflect` is the companion to `Proxy`. It provides the default implementations of all the operations Proxy can intercept. You use `Reflect` inside traps to "do the normal thing" after your custom logic — without it, you'd have to manually implement property assignment yourself.

```javascript
// ── Validation proxy ───────────────────────────────────────────────
const validator = {
  // `set` trap fires whenever a property is assigned: obj.prop = value
  set(target, prop, value) {
    if (prop === 'age') {
      if (typeof value !== 'number')
        throw new TypeError('Age must be a number');
      if (value < 0)
        throw new RangeError('Age must be non-negative');
    }
    // Reflect.set does the actual assignment — equivalent to target[prop] = value
    // but returns true/false instead of throwing, which is what the `set` trap expects
    return Reflect.set(target, prop, value);
  },
};

const person = new Proxy({}, validator);
person.name = 'Alice'; // fine — no validation on `name`
person.age = 30;       // fine
person.age = -1;       // RangeError: Age must be non-negative
person.age = 'old';    // TypeError: Age must be a number

// ── Reactive system (Vue 3-style) ──────────────────────────────────
// Whenever a property changes, notify subscribers.
function reactive(obj, onChange) {
  return new Proxy(obj, {
    set(target, key, value) {
      const result = Reflect.set(target, key, value); // do the real assignment
      onChange(key, value);   // then notify — this is how Vue triggers re-renders
      return result;
    },
  });
}

const state = reactive({ count: 0 }, (key, val) => {
  console.log(`${key} changed to ${val}`);
});
state.count = 1; // logs: "count changed to 1"
```

---

## 1.6 WeakMap, WeakSet & Symbol

### WeakMap

A regular `Map` holds strong references to its keys — as long as the Map exists, those keys (and their associated values) cannot be garbage collected, even if nothing else in the program references them. This can cause memory leaks when the map outlives the objects it's tracking.

`WeakMap` solves this: its keys must be objects, and it holds **weak references** to them. If an object used as a key is no longer reachable from anywhere else in the program, the garbage collector can reclaim it — and the WeakMap entry disappears automatically. This makes WeakMap ideal for associating private data with objects without preventing those objects from being collected.

```javascript
// ── Private data per instance ──────────────────────────────────────
// Before private class fields (#), this was the cleanest way to store
// data that should be inaccessible from outside the class.
const privateData = new WeakMap();

class User {
  constructor(name, age) {
    // Associate this instance with its private data.
    // When this User instance is garbage collected, this entry disappears too.
    privateData.set(this, { name, age });
  }
  getName() {
    return privateData.get(this).name;
  }
}

const user = new User('Alice', 30);
user.getName(); // "Alice"
// No way to access { name, age } from outside — it's not on `user` at all.
```

### WeakSet

`WeakSet` is similar: it holds weak references to objects, meaning membership doesn't prevent garbage collection. It's useful for tracking which objects have been "seen" or "processed" without keeping them alive.

```javascript
// Track which objects have been processed — won't leak if they go out of scope
const processed = new WeakSet();

function process(obj) {
  if (processed.has(obj)) return; // already done
  // ... do work ...
  processed.add(obj); // mark as done
}
```

### Symbol

`Symbol` creates a unique, non-string primitive value. Every call to `Symbol()` produces a value that is not equal to anything else, ever. This makes Symbols useful as object keys that are guaranteed not to collide with string keys or with each other — even if they have the same description.

```javascript
// ── Unique keys ────────────────────────────────────────────────────
const id = Symbol('id'); // 'id' is just a description for debugging
const user = {
  name: 'Alice',
  [id]: 42,         // Symbol as a computed property key
};

Object.keys(user);        // ['name'] — Symbol keys are excluded
JSON.stringify(user);     // '{"name":"Alice"}' — Symbols don't serialize
user[id];                 // 42 — accessible if you have a reference to the symbol

// Two Symbols with the same description are NOT equal:
Symbol('id') === Symbol('id'); // false

// ── Well-known Symbols: hooking into language protocols ────────────
// JavaScript uses specific Symbol keys internally to define behaviour.
// You can override them on your own objects.

// Symbol.iterator — makes an object work with for...of and spread
class Range {
  constructor(from, to) { this.from = from; this.to = to; }
  [Symbol.iterator]() {
    let current = this.from;
    const last = this.to;
    return { next: () => current <= last
      ? { value: current++, done: false }
      : { done: true } };
  }
}
[...new Range(1, 3)]; // [1, 2, 3]

// Symbol.toPrimitive — controls type coercion
class Money {
  constructor(amount, currency) { this.amount = amount; this.currency = currency; }
  [Symbol.toPrimitive](hint) {
    if (hint === 'number') return this.amount;   // used in +money, money * 2
    return `${this.amount} ${this.currency}`;    // used in `${money}`, "total: " + money
  }
}
+new Money(10, 'USD');          // 10
`${new Money(10, 'USD')}`;      // "10 USD"
```

---

## 1.7 ES Modules vs CommonJS

### Why Two Module Systems Exist

Node.js was created before JavaScript had a native module system, so it invented its own: CommonJS (`require` / `module.exports`). When ES2015 introduced native modules to the language (`import` / `export`), it had to coexist with the enormous existing Node.js ecosystem. That's why both exist today, and why understanding the differences matters for bundler configuration, tree-shaking, and interoperability.

The key practical difference: **ESM is static, CJS is dynamic**. ESM imports are resolved at parse time — the bundler can analyse the full module graph before any code runs, which enables tree-shaking (dead code elimination). CJS `require()` calls can be inside conditionals or loops, so the bundler can't know what will be loaded until runtime — making tree-shaking impossible.

```javascript
// ── ES Modules (ESM) ───────────────────────────────────────────────
// Imports are hoisted and resolved before any code runs.
// The bundler can statically determine exactly what you use.
import { add, PI } from './math.js';       // named import
import Calculator from './math.js';         // default import
import * as MathUtils from './math.js';     // namespace import

// Dynamic import — returns a Promise, used for code splitting / lazy loading
// This is the ESM way to load a module conditionally at runtime
const { add } = await import('./math.js');

// Exports
export const PI = 3.14;                    // named export
export function add(a, b) { return a + b; }
export default class Calculator {}         // default export (one per file)

// ── CommonJS (CJS) ─────────────────────────────────────────────────
// Synchronous — the module is loaded and executed when require() is called.
// `require` can be inside a function or if-block, making static analysis impossible.
const { add } = require('./math');
const mathUtils = require('./math');        // whole module as an object

module.exports = { add };                  // export an object
module.exports = Calculator;               // replace the whole export
```

| Feature | ESM | CJS |
|---|---|---|
| Static analysis | ✅ Yes | ❌ No |
| Tree-shaking | ✅ Yes | ❌ No |
| Top-level await | ✅ Yes | ❌ No |
| Circular deps | Live bindings | Partial at call time |
| Browser-native | ✅ Yes | ❌ No |
| Node.js native | ✅ Yes (v12+) | ✅ Yes |

**Interop:** `import()` (dynamic) works inside CJS to load ESM. `createRequire()` works inside ESM to load CJS.

---

## 1.8 Error Handling Patterns

### The Problem with Plain Errors

JavaScript's built-in `Error` class is minimal — it has a `message` and a `stack`, and nothing else. In production applications you almost always need more: an HTTP status code to send to the client, a machine-readable error code for logging, and a way to preserve the original cause when you wrap a low-level error in a higher-level one.

Beyond that, the standard `try/catch` pattern becomes painful in large codebases: errors thrown deep in a call stack unwind all the way up, requiring every layer to either catch and re-throw or let it bubble. The *Result pattern* addresses this by making errors explicit return values instead of exceptions — making error paths visible in the type system and eliminating surprise unwinds.

```typescript
// ── Custom error hierarchy ─────────────────────────────────────────
// A base class that adds a machine-readable `code` and HTTP `statusCode`.
// Every layer of the app throws AppError subclasses, not plain Errors.
class AppError extends Error {
  constructor(
    message: string,
    public readonly code: string,       // e.g. 'NOT_FOUND', 'DB_ERROR'
    public readonly statusCode = 500,   // HTTP status code for API responses
    options?: ErrorOptions,             // supports the `cause` option (ES2022)
  ) {
    super(message, options);
    this.name = this.constructor.name;  // "NotFoundError" instead of "Error"
    // V8-specific: removes this constructor from the stack trace,
    // so the trace starts where the error was thrown, not inside the constructor
    Error.captureStackTrace(this, this.constructor);
  }
}

// Specific error types inherit from AppError
class NotFoundError extends AppError {
  constructor(resource: string) {
    super(`${resource} not found`, 'NOT_FOUND', 404);
  }
}

class ValidationError extends AppError {
  constructor(message: string) {
    super(message, 'VALIDATION_ERROR', 400);
  }
}

// ── Error chaining with `cause` (ES2022) ───────────────────────────
// When you catch a low-level error and throw a higher-level one,
// attach the original as `cause` to preserve the full diagnostic chain.
// Both errors' stacks are available — you don't lose context.
async function getUser(id: string) {
  try {
    return await db.query('SELECT * FROM users WHERE id = $1', [id]);
  } catch (err) {
    // Wrap the DB error in an application-level error, but preserve the original
    throw new AppError('Failed to fetch user', 'DB_ERROR', 500, { cause: err });
  }
}
// In your error logger: err.cause.stack shows the original DB driver stack

// ── Result pattern (Railway-Oriented Programming) ──────────────────
// Instead of throwing, return a discriminated union: { ok: true, value }
// or { ok: false, error }. This makes error paths explicit in the type
// system — callers MUST handle the error case, or TypeScript will complain.
type Result<T, E = Error> =
  | { ok: true; value: T }
  | { ok: false; error: E };

async function parseConfig(text: string): Promise<Result<Config>> {
  try {
    return { ok: true, value: JSON.parse(text) as Config };
  } catch (err) {
    return { ok: false, error: new ValidationError(`Invalid config: ${(err as Error).message}`) };
  }
}

// At the call site, the error case is handled inline — no try/catch cascade
const result = await parseConfig(rawText);
if (!result.ok) {
  logger.error(result.error);
  return sendError(res, result.error);
}
// TypeScript now knows result.value is Config — fully typed, no cast needed
doSomethingWith(result.value);
```

---

## JavaScript Core Priority Summary

| Topic | Priority | Notes |
|---|---|---|
| Closures & scope | **Critical** | Asked in every senior interview |
| `this` binding rules | **Critical** | Common gotcha in callbacks & class methods |
| Prototype chain | **Deep** | Understand what `class` compiles to |
| Generators / async iterators | **Important** | Streaming, lazy sequences, coroutines |
| Proxy & Reflect | **Learn** | Vue/MobX reactivity under the hood |
| WeakMap / Symbol | **Know** | Memory-safe private patterns |
| ESM vs CJS | **Deep** | Tree-shaking, bundler behaviour |
| Error patterns + `cause` | **Important** | Production-quality error handling |

---


# 2. TypeScript Deep Dive

## Why TypeScript is Critical for Senior Roles

TypeScript adds a **structural static type system** on top of JavaScript. "Structural" means types are compared by their shape (what properties they have), not by their name or where they were declared — if two types have the same structure, they are compatible. "Static" means the compiler checks types at build time, before any code runs.

The benefits aren't just catching typos. TypeScript enables IDE autocompletion, safe refactoring across large codebases, and self-documenting APIs where the type signature tells you exactly what a function accepts and returns without reading the implementation. At a senior level, you're expected to express complex domain constraints in the type system itself — not just add `string` and `number` annotations.

90%+ of senior full-stack roles expect deep TypeScript knowledge. The sections below cover the patterns that come up in both daily work and technical interviews.

---

## 2.1 Advanced Types

### Mapped Types

A mapped type transforms every property of an existing type according to a rule. Instead of manually writing out each property, you iterate over the keys of a type and apply a transformation. This is how TypeScript's built-in `Partial`, `Readonly`, and `Record` are implemented internally.

The syntax `[P in keyof T]` iterates over every key in `T`, like a `for...in` loop but at the type level.

```typescript
// These are how the built-in utility types work under the hood:
type Readonly<T> = {
  readonly [P in keyof T]: T[P]; // add `readonly` to every property
};

type Partial<T> = {
  [P in keyof T]?: T[P]; // make every property optional
};

type Pick<T, K extends keyof T> = {
  [P in K]: T[P]; // keep only the keys listed in K
};

// ── Key remapping (advanced) ───────────────────────────────────────
// The `as` clause lets you rename keys during mapping.
// Here we create a type where every key becomes a getter method name.
type Getters<T> = {
  [K in keyof T as `get${Capitalize<string & K>}`]: () => T[K];
};

type User = { name: string; age: number };
type UserGetters = Getters<User>;
// Result: { getName: () => string; getAge: () => number }
// Useful for generating typed wrapper APIs, proxies, or ORMs.
```

### Conditional Types

A conditional type expresses "if T extends U, use X, otherwise use Y" — it's a ternary operator for types. This lets you write generic types that behave differently depending on what type they receive.

```typescript
type IsString<T> = T extends string ? true : false;
type A = IsString<string>;  // true
type B = IsString<number>;  // false

// `infer` inside a conditional type lets you capture a sub-type.
// Here we "unwrap" an array: if T is an array, extract the element type.
type Flatten<T> = T extends Array<infer U> ? U : T;
type Num = Flatten<number[]>; // number — inferred U from number[]
type Str = Flatten<string>;   // string — T wasn't an array, so returns T itself

// ── Distributive conditional types ────────────────────────────────
// When a conditional type receives a union, it distributes over each member.
// ToArray<string | number> becomes ToArray<string> | ToArray<number>
type ToArray<T> = T extends any ? T[] : never;
type StrOrNumArray = ToArray<string | number>; // string[] | number[]
```

### Template Literal Types

Template literal types build string types by composing other string types, using the same `` `${}` `` syntax as runtime template literals. They're useful for generating string union types from combinations of other unions, and for typing string-based APIs like event names or CSS properties.

```typescript
type HTTPMethod = "GET" | "POST";
type Route = "/users" | "/products";
// TypeScript computes every combination automatically
type Endpoint = `${HTTPMethod} ${Route}`;
// "GET /users" | "GET /products" | "POST /users" | "POST /products"

// ── Real-world use: typed event system ────────────────────────────
// This type ensures that event names are always `${keyName}Changed`,
// and that the callback receives the correct value type for that key.
type PropEventSource<T> = {
  on<K extends string & keyof T>(
    eventName: `${K}Changed`,      // only accepts "nameChanged", "ageChanged", etc.
    callback: (newValue: T[K]) => void,
  ): void;
};

declare function makeWatchedObject<T>(obj: T): T & PropEventSource<T>;

const person = makeWatchedObject({ name: "Alice", age: 30 });
person.on("nameChanged", (newName) => {}); // ✅ callback typed as (string) => void
person.on("ageChanged", (newAge) => {});   // ✅ callback typed as (number) => void
person.on("invalid", () => {});            // ❌ Type error — "invalid" is not a key
```

### Utility Types

TypeScript ships with a library of built-in generic types that cover the most common type transformations. Knowing these saves you from re-implementing them and signals TypeScript fluency to interviewers.

```typescript
type User = {
  id: number;
  name: string;
  email: string;
  password: string;
};

// Transform entire types
type PartialUser = Partial<User>;             // all properties optional
type RequiredUser = Required<User>;           // all properties required (remove ?)
type ReadonlyUser = Readonly<User>;           // all properties readonly

// Pick/exclude properties
type UserWithoutPassword = Omit<User, "password">;          // remove specific keys
type UserCredentials = Pick<User, "email" | "password">;    // keep only these keys

// Build types from scratch
type UserRecord = Record<string, User>;       // { [key: string]: User }
type RoleMap = Record<"admin" | "user", User[]>;

// Introspect functions
function getUser() { return { id: 1, name: "Alice" }; }
type UserShape = ReturnType<typeof getUser>;  // { id: number; name: string }
type GetUserArgs = Parameters<typeof getUser>; // [] (no parameters)

// Unwrap async types
type A = Awaited<Promise<string>>;            // string
type B = Awaited<Promise<Promise<number>>>;   // number (recursively unwrapped)
```

---

## 2.2 Generics & Constraints

### What are Generics?

Generics are TypeScript's way of writing code that works over many types while still preserving type information. Think of them as **type variables** — just like a function can have a value parameter `(x: number)`, a generic function can have a type parameter `<T>`. The caller provides the specific type when they use it.

Without generics, you'd have two bad options: write the same function multiple times for each type (duplication), or use `any` (which throws away all type safety). Generics give you the flexibility of `any` with the type safety of specific types.

```typescript
// Without generics — loses type info
function identity(arg: any): any { return arg; }
const result = identity(42); // result is `any`, not `number`

// With generics — T is inferred from the argument
function identity<T>(arg: T): T { return arg; }
const result = identity(42); // result is `number` ✅

// Generic functions work over any type:
function map<T, U>(arr: T[], fn: (item: T) => U): U[] {
  return arr.map(fn);
}
const lengths = map(['hello', 'world'], s => s.length); // number[]
// TypeScript infers T = string, U = number from usage
```

### Generic Constraints

Sometimes you want a generic to be flexible, but not completely unconstrained — you need to know it has certain properties. Constraints let you say "T can be any type, as long as it has this shape."

```typescript
// `T extends keyof U` means: T must be a key of U
// This lets TypeScript know that obj[key] is valid and type-safe
function getProperty<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key]; // TypeScript knows this is T[K], not `any`
}
const user = { name: 'Alice', age: 30 };
getProperty(user, 'name'); // string ✅
getProperty(user, 'xyz');  // ❌ Type error — 'xyz' is not a key of user

// Constraint via interface
interface HasLength { length: number; }
function logLength<T extends HasLength>(arg: T): T {
  console.log(arg.length); // safe — T is guaranteed to have .length
  return arg;
}
logLength('hello');   // ✅ string has .length
logLength([1, 2, 3]); // ✅ array has .length
logLength(42);        // ❌ number has no .length
```

### Generic Classes

Generics apply to class definitions the same way they do to functions — a type parameter makes the class reusable across entity types while preserving full type safety for every instance. A constraint (`T extends { id: number }`) lets you call methods on `T` that the compiler would otherwise reject.

```typescript
// A repository that works with any entity type
class GenericRepository<T extends { id: number }> {
  private items: T[] = [];

  add(item: T): void {
    this.items.push(item);
  }

  findById(id: number): T | undefined {
    // TypeScript knows item.id is valid because of the `T extends { id: number }` constraint
    return this.items.find(item => item.id === id);
  }

  findWhere(predicate: (item: T) => boolean): T[] {
    return this.items.filter(predicate);
  }
}

const userRepo = new GenericRepository<User>();
userRepo.add({ id: 1, name: 'Alice', email: 'a@example.com', password: 'x' });
const found = userRepo.findById(1); // User | undefined
```

---

## 2.3 Type Guards & Narrowing

### What is Narrowing?

TypeScript's type system gets more specific as it analyses code flow. When you check `typeof x === 'string'` inside an `if` block, TypeScript understands that `x` can only be `string` inside that block — it **narrows** the type from the broader union to the specific member. This is called control-flow analysis.

Narrowing is important because it lets you work with union types safely. Without it, you'd have to cast constantly. With it, TypeScript proves the type is safe within each branch.

### Built-in Narrowing Operators

TypeScript's control-flow analysis recognises specific operators and patterns that prove a type is narrower than declared. Each branch below tells the compiler exactly what type `x` must be, enabling you to call type-specific methods without a cast.

```typescript
// typeof — narrows primitive types
function format(x: string | number): string {
  if (typeof x === 'string') {
    return x.toUpperCase(); // TypeScript knows x is `string` here
  }
  return x.toFixed(2);     // TypeScript knows x is `number` here
}

// instanceof — narrows class instances
function processDate(value: Date | string) {
  if (value instanceof Date) {
    return value.toISOString(); // TypeScript knows value is `Date` here
  }
  return new Date(value).toISOString(); // TypeScript knows value is `string` here
}

// Truthiness narrowing — narrows out null/undefined
function greet(name: string | null) {
  if (name) {
    return `Hello, ${name}`; // name is `string` here — null was filtered
  }
  return 'Hello, stranger';
}
```

### Custom Type Guards

When `typeof` and `instanceof` aren't enough — for example, to distinguish between two plain object interfaces — you write a **type predicate** function. The return type `pet is Cat` is the predicate: it tells TypeScript "if this function returns true, the argument is of type Cat in the calling scope."

```typescript
interface Cat { meow: () => void; }
interface Dog { bark: () => void; }

// The `pet is Cat` return type is a type predicate.
// TypeScript will narrow `pet` to `Cat` in branches where this returns true.
function isCat(pet: Cat | Dog): pet is Cat {
  return 'meow' in pet; // runtime check — does this object have a `meow` property?
}

function makeSound(pet: Cat | Dog) {
  if (isCat(pet)) {
    pet.meow(); // TypeScript knows pet is `Cat` here ✅
  } else {
    pet.bark(); // TypeScript knows pet is `Dog` here ✅
  }
}
```

### Discriminated Unions

A discriminated union is a union of types that all share a common **literal** property (the discriminant). TypeScript uses that property to narrow to the exact member in a switch or if-else. This is the recommended pattern for modelling state — loading, success, error — in a type-safe way.

```typescript
// Each variant has a unique literal `status` — the discriminant
type LoadingState = { status: 'loading' };
type SuccessState = { status: 'success'; data: User[] };
type ErrorState   = { status: 'error';   error: string };

type RequestState = LoadingState | SuccessState | ErrorState;

function render(state: RequestState) {
  switch (state.status) {
    case 'loading':
      return <Spinner />;            // state is `LoadingState` — no `.data` available
    case 'success':
      return <UserList users={state.data} />; // state is `SuccessState` — .data is string[]
    case 'error':
      return <ErrorMessage msg={state.error} />; // state is `ErrorState`
  }
  // TypeScript knows the switch is exhaustive — no default needed
}
```

---

## 2.4 Advanced Patterns

### The `infer` Keyword

`infer` appears inside conditional types and means "extract this type and give it a name." It's how you pull a type out of another type's structure — like destructuring, but for types.

The classic use cases: extracting the return type of a function, the element type of an array, or the resolved value of a Promise. TypeScript's built-in `ReturnType<T>`, `Parameters<T>`, and `Awaited<T>` are all implemented with `infer`.

```typescript
// Extract what a function returns — equivalent to TypeScript's built-in ReturnType<T>
// "If T is a function type, extract its return type as R, otherwise never"
type MyReturnType<T> = T extends (...args: any[]) => infer R ? R : never;

function getUser() { return { id: 1, name: 'Alice' }; }
type User = MyReturnType<typeof getUser>; // { id: number; name: string }

// Extract the element type of an array
type UnpackArray<T> = T extends (infer U)[] ? U : T;
type Item = UnpackArray<string[]>; // string
type Same = UnpackArray<number>;   // number (not an array, falls through to T)

// Extract the resolved type of a Promise (simplified version of Awaited<T>)
type UnpackPromise<T> = T extends Promise<infer U> ? U : T;
type Value = UnpackPromise<Promise<number>>; // number
```

### Branded Types (Nominal Typing)

TypeScript uses structural typing — two types are compatible if they have the same shape. This is usually good, but sometimes it causes bugs: a `UserId` and a `ProductId` are both `number`, so you can accidentally pass one where the other is expected and TypeScript won't catch it.

Branded types solve this by adding a phantom type tag that makes structurally identical types incompatible. The tag only exists at the type level — at runtime, it's still just a number.

```typescript
// Create a unique brand for each domain ID type.
// `unique symbol` ensures each brand is truly distinct, even if they look the same.
type UserId    = number & { readonly __brand: unique symbol };
type ProductId = number & { readonly __brand: unique symbol };

// Constructor functions validate and cast to the branded type
function createUserId(id: number): UserId {
  if (id <= 0) throw new Error('Invalid user ID');
  return id as UserId;
}

function getUser(id: UserId): User { /* ... */ }

const userId    = createUserId(123);
const productId = 456 as ProductId;

getUser(userId);    // ✅ correct
getUser(productId); // ❌ Type error: ProductId is not assignable to UserId
getUser(789);       // ❌ Type error: number is not assignable to UserId

// At runtime: userId is still just 123 — no overhead
```

### Builder Pattern with Fluent API

The Builder pattern separates the construction of a complex object from its representation. Each method returns `this`, enabling a fluent chain that reads naturally and composes incrementally — the final `build()` or `execute()` call produces the result. TypeScript's `this` return type preserves the concrete subclass type through the chain so it works correctly with inheritance.

```typescript
// The builder accumulates filters and executes them.
// `this` return type enables method chaining with correct TypeScript inference
// even for subclasses — a subclass's .where() returns the subclass type, not QueryBuilder.
class QueryBuilder<T> {
  private filters: Array<(item: T) => boolean> = [];

  where(predicate: (item: T) => boolean): this {
    this.filters.push(predicate);
    return this; // return `this` (not `QueryBuilder<T>`) for chainability in subclasses
  }

  execute(data: T[]): T[] {
    return data.filter(item => this.filters.every(f => f(item)));
  }
}

const results = new QueryBuilder<User>()
  .where(u => u.age > 18)
  .where(u => u.name.startsWith('A'))
  .execute(users);
```

---

## 2.5 TypeScript 5.x Features

### `const` Type Parameters (5.0)

Normally, TypeScript widens array literals to `T[]` when inferring a generic parameter. `const` on a type parameter tells TypeScript to infer the narrowest possible type — treating the array as if it were declared `as const`.

```typescript
// Without `const`: infers string[], losing the literal types
function makeArray<T>(arr: T[]): T[] { return arr; }
const a = makeArray(['red', 'green']); // string[] — literals lost

// With `const`: infers readonly ["red", "green"] — preserves literals
function makeArray<const T>(arr: T[]): T[] { return arr; }
const b = makeArray(['red', 'green']); // readonly ["red", "green"] ✅
```

### `satisfies` Operator (4.9+)

The `satisfies` operator validates that a value matches a type **without widening** the value's type. With a plain type annotation (`const x: Type = value`), TypeScript widens `x` to `Type`. With `satisfies`, the annotation is just a check — `x` keeps its original narrow type.

```typescript
type Color = { r: number; g: number; b: number } | string;

// With annotation: palette.red is `Color` — you lose the array methods
const palette1: Record<string, Color> = {
  red: [255, 0, 0],
  green: '#00ff00',
};
palette1.red[0]; // ❌ Property '0' does not exist on type 'Color'

// With satisfies: TypeScript checks the shape, but keeps the narrow types
const palette2 = {
  red: [255, 0, 0],    // inferred as number[]
  green: '#00ff00',    // inferred as string
} satisfies Record<string, Color>;

palette2.red[0];  // ✅ number — TypeScript knows red is number[], not just Color
palette2.green.toUpperCase(); // ✅ string — TypeScript knows green is string
```

### Decorators (5.0 — Stage 3)

Decorators are a meta-programming feature that let you annotate and modify classes and class members. TypeScript 5.0 implemented the TC39 Stage 3 proposal, which has different semantics from the older experimental decorators. They are heavily used in frameworks like NestJS and Angular.

```typescript
// A decorator is a function that receives the class or member it decorates.
// This decorator wraps a method to log calls to it.
function logged(target: any, context: ClassMethodDecoratorContext) {
  const methodName = String(context.name);
  return function (this: any, ...args: any[]) {
    console.log(`Calling ${methodName} with`, args);
    return target.apply(this, args); // call the original method
  };
}

class Calculator {
  @logged
  add(a: number, b: number) {
    return a + b;
  }
}

new Calculator().add(2, 3);
// logs: "Calling add with [2, 3]"
// returns: 5
```

---

## 2.6 Configuration Best Practices

### `tsconfig.json` — What Each Flag Does

`strict: true` is a shorthand for six stricter checks: `strictNullChecks`, `strictFunctionTypes`, `strictBindCallApply`, `strictPropertyInitialization`, `noImplicitAny`, and `noImplicitThis`. These catch the most common categories of runtime bugs at compile time. You should always enable `strict` in new projects.

The flags below go further and are recommended for production codebases:

```json
{
  "compilerOptions": {
    "strict": true,
    // `noUncheckedIndexedAccess`: arr[i] returns T | undefined, not T.
    // Forces you to handle the case where the index is out of bounds.
    "noUncheckedIndexedAccess": true,

    // `noImplicitOverride`: subclass methods that shadow parent methods
    // must be explicitly marked `override`. Prevents accidental shadowing.
    "noImplicitOverride": true,

    // `exactOptionalPropertyTypes`: distinguishes `{ x?: string }` (key may be absent)
    // from `{ x: string | undefined }` (key present but value is undefined).
    "exactOptionalPropertyTypes": true,

    // `noFallthroughCasesInSwitch`: every switch case must end with break/return/throw.
    "noFallthroughCasesInSwitch": true,

    // `noImplicitReturns`: every code path in a function must return a value.
    "noImplicitReturns": true,

    // Build / module settings for a modern full-stack project:
    "skipLibCheck": true,           // skip type-checking node_modules (faster builds)
    "esModuleInterop": true,        // allows `import React from 'react'` style imports
    "moduleResolution": "bundler",  // matches how Vite/webpack resolve modules
    "module": "ESNext",
    "target": "ES2022",
    "lib": ["ES2022", "DOM", "DOM.Iterable"]
  }
}
```

---

## TypeScript Priority Summary

| Topic | Priority | Notes |
|---|---|---|
| Advanced types (mapped, conditional, template literal) | **Critical** | Appear in every senior TS codebase |
| Generics + constraints | **Critical** | Core tool for reusable, type-safe code |
| Type guards, discriminated unions | **Critical** | How you work safely with union types |
| Utility types (Partial, Pick, Omit, Record, etc.) | **Deep** | Know all of them without looking up |
| `infer` keyword | **Learn** | Needed for advanced library types |
| Branded types | **Know** | Prevents ID mix-up bugs |
| TypeScript 5.x features (`const` params, `satisfies`) | **Learn** | Increasingly common in new codebases |
| `tsconfig.json` strict mode | **Deep** | Know what each flag enables and why |

---


# 3. Async and Event-Driven Programming

## Why Async Matters

JavaScript is **single-threaded**: there is one call stack, and only one piece of code runs at a time. If that were the whole story, any slow operation — reading a file, making an HTTP request, waiting for a database query — would freeze the entire program. The event loop is how JavaScript avoids this while staying single-threaded.

Understanding the event loop is important not just for avoiding bugs (like the `var` in loops pitfall), but for reasoning about execution order when mixing `setTimeout`, Promises, and `async/await`. Senior interviews frequently include "what order do these log?" questions that require a mental model of the event loop.

---

## 3.1 The Event Loop

### What It Is

The event loop is a continuous loop that:
1. Runs all synchronous code to completion (the current "tick")
2. Drains the **microtask queue** completely (Promises, `queueMicrotask`, `process.nextTick`)
3. Picks one task from the **macrotask queue** (setTimeout, setInterval, I/O callbacks)
4. Repeats

The key insight: **microtasks always run before the next macrotask**. This means Promise callbacks have higher priority than `setTimeout` callbacks, even with a 0ms delay.

### Node.js Event Loop Phases

Node.js has a more structured event loop than the browser, divided into phases managed by the underlying C library (libuv):

```
   ┌───────────────────────────┐
┌─>│           timers          │  setTimeout, setInterval callbacks run here
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │     pending callbacks     │  I/O error callbacks from the previous iteration
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │           poll            │  retrieve new I/O events; execute their callbacks
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │           check           │  setImmediate callbacks run here (after poll)
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │      close callbacks      │  e.g. socket.on('close', ...)
│  └───────────────────────────┘
└────────────────────────────────

Between EVERY phase transition — and after each individual callback:
  1. process.nextTick() queue (highest priority — runs before anything else)
  2. Promise microtask queue
```

**Priority order (highest to lowest):**
1. `process.nextTick()` — runs before any other async callback, even other microtasks
2. Promise microtasks (`.then`, `.catch`, `await` continuations)
3. `setImmediate` — runs in the "check" phase, after I/O poll
4. `setTimeout(fn, 0)` — runs in the "timers" phase on the next loop iteration

```javascript
// Classic execution order question:
console.log('1');                          // sync — runs immediately

setTimeout(() => console.log('2'), 0);     // macrotask — timers phase

Promise.resolve().then(() => console.log('3')); // microtask

process.nextTick(() => console.log('4'));  // nextTick — highest priority async

console.log('5');                          // sync — runs immediately

// Output: 1, 5, 4, 3, 2
// Sync runs first (1, 5), then nextTick (4), then Promise microtask (3),
// then macrotask (2)
```

---

## 3.2 Async Patterns

### Promise Combinators

The four Promise combinators (`all`, `allSettled`, `race`, `any`) differ in how they handle failures and which result they return. Picking the right one for a situation is a common interview topic.

```typescript
// ── Promise.all — run in parallel, fail immediately if any fails ───
// Use when: all results are needed and any failure should abort the whole operation.
// If any promise rejects, the whole thing rejects immediately (others keep running but results are ignored).
const [users, posts, comments] = await Promise.all([
  fetchUsers(),
  fetchPosts(),
  fetchComments(),
]);
// All three fetches start simultaneously — much faster than awaiting sequentially

// ── Promise.allSettled — run in parallel, wait for all regardless ──
// Use when: you need results from all promises even if some fail.
// Returns an array of { status: 'fulfilled', value } or { status: 'rejected', reason }
const results = await Promise.allSettled([
  fetchUsers(),
  fetchPosts(),
  fetchComments(),
]);

results.forEach((result) => {
  if (result.status === 'fulfilled') {
    console.log('Success:', result.value);
  } else {
    console.error('Failed:', result.reason); // handle partial failures gracefully
  }
});

// ── Promise.race — first to settle wins (fulfilled OR rejected) ────
// Use when: you want the fastest response and don't care about the others.
// Classic use case: timeout — race the actual request against a timeout promise.
const response = await Promise.race([
  fetch('https://api.example.com/data'),
  new Promise((_, reject) => setTimeout(() => reject(new Error('Timeout')), 5000)),
]);

// ── Promise.any — first to fulfil wins (ignores rejections) ────────
// Use when: you have multiple sources and want whichever succeeds first.
// Unlike race(), it ignores rejections unless ALL reject (throws AggregateError).
const fastestOk = await Promise.any([
  fetch('https://cdn1.example.com/data'),
  fetch('https://cdn2.example.com/data'),
]);
```

### Retry with Exponential Backoff

When a network request fails, retrying immediately usually fails for the same reason (the server is overloaded, a transient DNS issue, etc.). Exponential backoff increases the delay between retries exponentially — this gives the upstream service time to recover and avoids thundering herd problems where thousands of clients all retry at the same moment.

```typescript
async function retry<T>(
  fn: () => Promise<T>,
  maxRetries = 3,
  baseDelayMs = 1000,
  backoffMultiplier = 2,
): Promise<T> {
  let lastError: Error;

  for (let attempt = 0; attempt < maxRetries; attempt++) {
    try {
      return await fn(); // try the operation
    } catch (err) {
      lastError = err as Error;

      if (attempt < maxRetries - 1) {
        // Delay grows: 1000ms, 2000ms, 4000ms, ...
        // Adding jitter (Math.random()) prevents all clients retrying simultaneously
        const delay = baseDelayMs * Math.pow(backoffMultiplier, attempt);
        await new Promise(resolve => setTimeout(resolve, delay));
      }
    }
  }

  throw lastError!; // all retries exhausted
}

const data = await retry(
  () => fetch('https://api.example.com/data').then(r => r.json()),
  5,      // up to 5 attempts
  500,    // starting delay: 500ms
);
```

### AbortController

`fetch` has no built-in timeout. Without `AbortController`, a hung request would keep the Promise pending forever, potentially causing resource leaks and degraded user experience. `AbortController` lets you cancel any `fetch` (or any API that accepts an `AbortSignal`) programmatically.

```typescript
async function fetchWithTimeout(url: string, timeoutMs: number) {
  const controller = new AbortController();

  // After `timeoutMs`, send the abort signal to any listener
  const timeoutId = setTimeout(() => controller.abort(), timeoutMs);

  try {
    const response = await fetch(url, {
      signal: controller.signal, // pass the signal to fetch
    });
    const data = await response.json();
    clearTimeout(timeoutId); // cancel the timeout — we succeeded
    return data;
  } catch (err) {
    if ((err as Error).name === 'AbortError') {
      throw new Error(`Request timed out after ${timeoutMs}ms`);
    }
    throw err; // re-throw non-abort errors
  }
}

// AbortController can also cancel in-flight requests when a component unmounts
// (a common React pattern to avoid setState on unmounted components)
useEffect(() => {
  const controller = new AbortController();
  fetch('/api/data', { signal: controller.signal }).then(setData);
  return () => controller.abort(); // cleanup: cancel the request if component unmounts
}, []);
```

---

## 3.3 Event-Driven Architecture

### What is Event-Driven Architecture?

In a traditional request-response architecture, services call each other directly: Service A calls Service B's API, waits for the response, then continues. This creates **tight coupling** — A knows about B's interface, A can't continue until B responds, and if B is slow or down, A is affected.

Event-Driven Architecture (EDA) decouples services through events. Service A emits an event ("something happened") and has no knowledge of what, if anything, happens next. Other services subscribe to events they care about and react independently. This means:
- Services can evolve independently
- New subscribers can be added without changing the producer
- Failures in subscribers don't affect the producer
- You get a natural audit log of everything that happened

### Event Notification

The simplest EDA pattern. The event just announces that something happened — it carries minimal data (typically just an ID). Subscribers query for more details if they need them.

```typescript
// Producer: knows nothing about who consumes the event or what they do
class UserService {
  async createUser(data: CreateUserDto) {
    const user = await this.db.users.create(data);
    // Emit a lightweight notification — just the ID
    this.eventBus.emit('user.created', { userId: user.id });
    return user;
  }
}

// Consumer 1: decides independently to send a welcome email
eventBus.on('user.created', async ({ userId }) => {
  const user = await userService.findById(userId); // fetches details it needs
  await emailService.sendWelcome(user.email);
});

// Consumer 2: decides independently to track the signup
eventBus.on('user.created', async ({ userId }) => {
  await analyticsService.trackSignup(userId);
});

// Adding a new consumer (e.g. Slack notification) requires zero changes to UserService
```

### Event-Carried State Transfer

A variation where the event carries all the data consumers need, so they don't have to query back to the producer. This trades larger event payloads for fewer inter-service queries — useful when consumers need multiple fields and querying them individually would create tight coupling or performance issues.

```typescript
class UserService {
  async createUser(data: CreateUserDto) {
    const user = await this.db.users.create(data);
    // Event carries the full snapshot of the created user
    this.eventBus.emit('user.created', {
      userId: user.id,
      email: user.email,
      name: user.name,
      createdAt: user.createdAt,
    });
    return user;
  }
}

// Consumer doesn't need to call UserService at all — data is in the event
eventBus.on('user.created', async (userData) => {
  await emailService.sendWelcome(userData.email, userData.name);
  // No extra network call — everything needed is already in the event payload
});
```

### CQRS (Command Query Responsibility Segregation)

CQRS separates the **write model** (commands that change state) from the **read model** (queries that read state). Instead of a single data model that serves both reads and writes, you have two: a write side that handles commands and emits events, and a read side with denormalised projections optimised for fast queries.

This matters at scale because read patterns and write patterns are often incompatible. Writes need ACID guarantees and normalised data. Reads need speed and pre-joined, pre-aggregated data. CQRS lets you optimise each independently.

```typescript
// ── Write side: handles commands, stores events ────────────────────
// A "command" expresses intent to change state.
// The handler loads the aggregate, applies the change, and saves the events.
@CommandHandler(DepositMoneyCommand)
class DepositMoneyHandler {
  async execute(command: DepositMoneyCommand) {
    const account = await this.eventStore.load(command.accountId);
    account.deposit(command.amount); // mutates the aggregate, records a domain event
    await this.eventStore.save(account); // persists the new events
    // No return value — commands don't return data
  }
}

// ── Read side: projection (denormalised view) ──────────────────────
// Listens for events and maintains a flat, query-optimised read table.
// This table can be rebuilt at any time by replaying the event stream.
@EventsHandler(MoneyDepositedEvent)
class AccountBalanceProjection {
  async handle(event: MoneyDepositedEvent) {
    // Update the read model — this table is optimised for fast balance lookups
    await this.db.query(
      `UPDATE account_balances SET balance = balance + $1 WHERE account_id = $2`,
      [event.amount, event.accountId],
    );
    // The read model is eventually consistent — it updates asynchronously after writes
  }
}
```

---

## Async & EDA Priority Summary

| Topic | Priority | Notes |
|---|---|---|
| Event loop phases + microtask vs macrotask | **Deep** | Execution order questions in every senior interview |
| `process.nextTick` vs Promise vs setTimeout order | **Deep** | Node.js-specific, frequently asked |
| `Promise.all` / `allSettled` / `race` / `any` | **Critical** | When to use each is a common question |
| Retry with exponential backoff | **Deep** | Production async pattern |
| `AbortController` | **Learn** | Increasingly expected in modern code |
| Event-Driven Architecture (notification vs state transfer) | **Important** | Architecture design questions |
| CQRS | **Important** | System design and NestJS context |

---

---

_End of Part 1. Continue to **Part 2** (Frontend) in [`02_Frontend_4-15.md`](./02_Frontend_4-15.md), or return to the [README](./README.md)._
