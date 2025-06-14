# Progressive JSON – Design Document

> [!NOTE]
> The content and technical details of this document have been carefully reviewed for accuracy. 
> While the wording and phrasing have been generated with the assistance of AI, the core concepts, data, and technical information are sound and accurate.

## Goal

Progressive JSON defines a protocol for incrementally constructing a single JSON object using a sequence of **non-overlapping fragments**, each representing a pratial update. This pattern enables partial data delivery, allowing systems to emit or consume parts of a document incrementally.

This document defines:

* The **rules and constraints** for building a valid Progressive JSON stream
* The **contract between server and client**
* Guidelines for **handling types and intermediate states**
* Recommendations for testing and validation

The pattern can be implemented by any client or server regardless of programming language or transport layer.

## Overview

Progressive JSON allows a server to emit a stream of **partial JSON objects** (called *fragments*) that, when combined, form a single complete object. Each fragment must be strictly additive:

* **No value may be overwritten**
* **Arrays must be extended**, not replaced

Once merged, the resulting JSON object reflects all the received fragments, assembled into a single tree.

This model is useful in situations where:

* Different parts of the data become available at different times
* Clients can benefit from rendering or acting on partial data early
* The full response might be large, and partial progress is meaningful

## Responsibilities

### Client

The **client library** is responsible for:

* Receiving and parsing each JSON fragment
* Merging it into the evolving JSON result
* Ensuring no conflicts occur (no overwrites, no type changes)
* Making intermediate and final results available to consumers (e.g., application code, UI)

The client **does not**:

* Attempt to infer missing fields
* Perform schema validation
* Retry or re-request specific fragments
* Decide how data is chunked or emitted

### Server

The **server library** is responsible for:

* Computing and emitting fragments in a valid, non-conflicting way
* Ensuring that fields are never redefined or overwritten across fragments
* Emitting array items in the intended final order

The server has full freedom in how fragments are produced, it may use:

* Local data immediately available
* Asynchronous operations or remote calls
* Deferred computation or event-driven logic
* Any transport (HTTP streaming, WebSocket, file system, etc.)

#### Incremental Construction

The server-side logic **must be structured to incrementally construct the final result**, rather than assembling it all at once.

That means the developer should be able to decompose the construction of the full JSON object into **independent operations**, each contributing a part of the final result. Each operation's output must be provided in a way that the library can process and emit as a valid Progressive JSON fragment.

This may look different depending on the language:

* **In JavaScript/TypeScript**:
  * Each asynchronous task can return a partial object.
  * The library can merge this partial into the current result and compute a fragment by comparing the previous and new intermediate state.
* **In C#**:
  * A shared intermediate object may be passed to all contributing operations.
  * Each operation can mutate this object directly.
  * The library can capture a diff between snapshots of this object before and after mutation to produce a fragment.

The key requirement is that the final object is not built monolithically, but instead **constructed from independent, additive steps**, each of which produces a valid fragment. This allows the server library to progressively emit valid fragments that can be consumed by the client.

## Use Case Example

Imagine an API endpoint that returns an invoice with supplementary data retrieved on demand.

The server might emit:

1. Initial invoice metadata:

   ```json
   {
     "id": "INV-1234",
     "total": 150.00,
     "date": "2025-06-10"
   }
   ```

2. Later, after an async call:

   ```json
   {
     "paymentStatus": {
       "state": "Paid",
       "paidAt": "2025-06-12T08:15:00Z"
     }
   }
   ```

3. Still later:

   ```json
   {
     "comments": [
       { "author": "Bob", "message": "The payment has been confirmed." },
       { "author": "Charlie", "message": "The invoice needs to be sent to the customer." }
     ]
   }
   ```

The client simply merges these in order, resulting in a single coherent object.

## Core Rules

### Fragments

A **fragment** is a valid JSON object that:

* Contributes new fields or subfields
* Does **not overwrite** any previously set data
* May define array content that should be appended

Examples:

* Fragment 1:

  ```json
  { "user": { "name": "Alice" } }
  ```

* Fragment 2:

  ```json
  { "user": { "age": 30 } }
  ```

* Result:

  ```json
  { "user": { "name": "Alice", "age": 30 } }
  ```

But the following is invalid if applied **after** Fragment 1:

```json
{ "user": { "name": "Bob" } }
```

→ Conflict: `user.name` already set.

### Constraints Summary

| Constraint        | Description                                                                 |
| ----------------- | --------------------------------------------------------------------------- |
| No Overwrites     | A scalar field may not be set more than once                              |
| No Type Mutation  | A field may not change type after being set (e.g., object → scalar)         |
| Array Append Only | Arrays can only grow, they cannot be replaced or re-ordered                |

### Array Behavior

Fragments may append to arrays but may not redefine them:

* Valid:

  ```json
  { "items": [1, 2] }
  ```

  then

  ```json
  { "items": [3] }
  ```

  → Result: `{ "items": [1, 2, 3] }`

The order of array items is defined **by the server**, and the client will **preserve order as received**.

## Typing and Intermediate Results

In statically typed environments, fragment-based construction results in **incomplete intermediate states**. Until all fragments are received, the structure may not fully match the final schema. Different strategies exist to handle this partial state depending on the language and tooling available.

### Strategy 1: Intermediate Types

A structured approach involves defining an **intermediate type** that represents the partial shape of the final result. This type allows fields to be missing or undefined during progressive construction, providing better tooling and compile-time checks.

#### In TypeScript

Intermediate types can be defined generically using recursive type utilities:

```ts
type RecursivePartial<T> = {
  [P in keyof T]?:
    T[P] extends (infer U)[] ? RecursivePartial<U>[] :
    T[P] extends object ? RecursivePartial<T[P]> :
    T[P];
};
```

This allows constructs like `RecursivePartial<T>` to describe the evolving state with optional fields.

#### In C#

Intermediate types can often be **automatically generated** from the final result type using source generators, reflection, or manual definitions with nullable fields. This enables:

* Strong typing throughout construction
* IntelliSense and compile-time checks
* Optional validation at intermediate stages

#### Other Languages

In languages that lack metaprogramming or flexible generics, defining an intermediate type may require significant boilerplate. In such cases, using raw objects may be the more pragmatic solution.

### Strategy 2: Use Raw Objects

An alternative approach is to represent intermediate state using untyped structures:

* Use untyped representations such as `object`, `Map<String, Object>`, or equivalent.
* Merge fragments into this structure without enforcing a concrete schema.
* Once all fragments are received, deserialize the final merged result into the expected type.

This approach minimizes boilerplate and simplifies merging logic, though it sacrifices type safety during intermediate stages.

## Not Production-Oriented

**Progressive JSON is a fun experimentation and not recommended for production**, most real-world use cases are better served by **separate API calls** for distinct data because it offers better reliability, scalability, and maintainability.

## Test cases for client libraries

Simple test cases are provided in the [`tests/`](tests/) folder.

Test cases should validate client behavior by applying a sequence of fragments.

Each test consists of:

* A list of `fragment` steps
* A `success` boolean per step
* A `currentResult` object (expected full state after merge)

Test logic should:

1. Apply each fragment in order
2. If `success: true`, assert no error and compare to expected `currentResult`
3. If `success: false`, assert an error occurred

A test schema is provided at:
[`tests/progressive-json-test-case.schema.json`](tests/progressive-json-test-case.schema.json)

## Summary

Progressive JSON defines a contract for **additive, conflict-free merging** of JSON fragments. The **server** has full control over how and when fragments are computed. The **client** only needs to enforce constraints and maintain a valid, evolving view of the JSON object.

This model is:

* Useful for **incremental rendering** or **data previews**
* Compatible with any language or transport
* Easy to reason about due to strict constraints

However, simpler alternatives exist for most real-world scenarios.