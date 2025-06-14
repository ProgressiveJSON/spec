# Progressive JSON – Client Library Design Document

> [!NOTE]
> The content and technical details of this document have been carefully reviewed for accuracy. 
> While the wording and phrasing have been generated with the assistance of AI, the core concepts, data, and technical information are sound and accurate.

## Goal

This design document outlines the concepts for building a **progressive JSON client** that can be implemented in any programming language. The goal is to allow developers to create client libraries in their preferred languages, while ensuring that they can seamlessly interact with server implementations built in other languages.

The library allows for **progressively building a single JSON object** using a stream of **non-overlapping JSON fragments**. Each fragment contributes to the whole, but **no part of the structure may be overwritten once set**.

It is important to note that this library is **not about editing a JSON document over time**. Instead, it is about receiving pieces of a document that was **always meant to be assembled this way**: each fragment is part of a well-defined, incremental process of building the final document.

The design focuses on:

* **Parsing** each JSON fragment
* **Merging** it into the resulting JSON object
* **Exposing** the current result and final result

## Use Case

Imagine an application that serves invoice documents through an API. When a user requests an invoice, the server must assemble a response that includes:

* The **invoice data** (e.g. ID, line items, total): available immediately from local storage or cache.
* The **payment status**: fetched asynchronously from a remote payment provider's API.

### With Progressive JSON

Using Progressive JSON, the server can:

1. Immediately stream the invoice details:
   ```json
   {
     "id": "INV-1234",
     "total": 150.00,
     "date": "2025-06-10",
   }
   ```
2. Then later, stream the payment status:
   ```json
   {
     "paymentStatus": {
       "state": "Paid",
       "paidAt": "2025-06-12T08:15:00Z"
     }
   }
   ```
3. Then later, stream the comments of users:
   ```json
   {
     "comments": [
      {
        "author": "Bob",
        "message": "The payment has been confirmed."
      },
      {
        "author": "Charlie",
        "message": "The invoice needs to be sent to the customer."
      }
    ]
   }
   ```

The client progressively builds the final object and exposes the merged result to the application or UI.

### But wait... Isn't this overkill?

This example illustrates the core concept well, but it's also worth noting that in real-world systems, **simpler and more robust alternatives often exist**:

1. **Separate APIs**
   Have one API to fetch invoice data, another to fetch the payment status and another to fetch the comments. This makes each responsibility explicit and allows clients to handle latency, errors, and retries independently.
   For simple uses cases like this, this is the most reasonnable approach.

2. **Background Processing**
   The server could fetch payment statuses asynchronously (e.g., using webhooks or scheduled polling) and store them in a local database. This way, the full data is always available instantly when the invoice API is called, no streaming needed.

## Core Concepts

### Fragment

A **fragment** is a standalone JSON object that contains a subset of the final structure.

Example:

- Fragment 1
```json
{ "user": { "name": "Alice" } }
```

- Fragment 2
```json
{ "user": { "age": 30 } }
```

- Result
```json
{ "user": { "name": "Alice", "age": 30 } }
```

If a later fragment attempts to redefine an already set field, such as `user.name`, it will be rejected as a **conflict**.

### Merge Constraints

* **No overwrites allowed**
  Once a field has been set, it cannot be overwritten, including:
  * Changing a scalar value
  * Replacing an object or array with another type

* **Arrays must be appended**
  Arrays grow over time; their content may arrive in separate fragments, but the same array should never be redefined. The server must emit array elements in the correct order: the builder only appends and does not sort or deduplicate.\
  Example:
  - Fragment 1: `{ "comments": ["Hello there!", "How are you?"] }`
  - Fragment 2: `{ "comments": ["Fine thanks!"] }`
  - Result: `{ "comments": ["Hello there!", "How are you?", "Fine thanks!"] }`

## Design Goals

* Accept JSON fragments that incrementally extend the current result without conflicts
* Identify and reject overlapping or conflicting paths
* Expose the evolving object to consumers in real-time
* Ensure transport-agnostic design, with no assumptions about the data source (e.g., not limited to HTTP responses)

Specifically, the library does **not** aim to:
* Support overwrites, diffs, or patches
* Enforce schemas or strict typing, beyond basic type consistency
* Make assumptions about the timing, order, or completion of fragments

## API Sketch (C#-style)

```csharp
/// <summary>
/// Builds a single immutable JSON object from a sequence of non-overlapping JSON fragments.
/// </summary>
/// <remarks>
/// The builder accepts JSON fragments that extend the current object without overwriting any previously defined fields.
/// Fragments must be additive and non-conflicting. Scalars cannot be overwritten, and objects or arrays must not be redefined.
/// Designed for use in streaming or progressive response scenarios.
/// </remarks>
public class ProgressiveJsonBuilder
{
    /// <summary>
    /// Applies a JSON fragment to the current object.
    /// </summary>
    /// <param name="jsonFragment">
    /// The JSON fragment to apply, as a string. It must represent a JSON object and must not overlap with previously applied data.
    /// </param>
    /// <exception cref="InvalidOperationException">
    /// Thrown if the fragment attempts to overwrite an existing value, replace a scalar, or change the type of an existing object or array.
    /// </exception>
    /// <remarks>
    /// Triggers <see cref="OnUpdate"/> if successful, or <see cref="OnError"/> if an error occurs.
    /// </remarks>
    public void ApplyFragment(string jsonFragment) { ... }

    /// <summary>
    /// Gets the current state of the JSON object after applying all fragments so far.
    /// </summary>
    /// <returns>The current reconstructed <see cref="JsonObject"/>.</returns>
    /// <remarks>
    /// The returned <see cref="JsonObject"/> is immutable. It can be safely used throughout the application,
    /// even if new fragments are applied after it is retrieved. Any updates made to the builder will not 
    /// affect the previously retrieved object.
    /// </remarks>
    public JsonObject GetCurrentJson() { ... }
}
```

## Intermediate Results and Typing

One of the key challenges with Progressive JSON is that **intermediate results** are often **incomplete** until the last fragment is received. This means that during the process of progressively merging the fragments, the structure of the JSON may not fully match its final schema or type.

For example, in a TypeScript application, the **final result** might be of type `T`, but the **intermediate results** would have the type `Partial<T>`, or more specifically a special `RecursivePartial<T>` that represents the type where any nested field can be `undefined`. 

```typescript
type RecursivePartial<T> = {
  [P in keyof T]?:
    T[P] extends (infer U)[] ? RecursivePartial<U>[] :
    T[P] extends object | undefined ? RecursivePartial<T[P]> :
    T[P];
};
```
(see [this stackoverflow post](https://stackoverflow.com/questions/41980195/recursive-partialt-in-typescript))

This ensures that intermediate results can accommodate missing or incomplete fields, without violating the type system.

For static-typed languages like C# or Java, the transition from dynamically typed data (used for intermediate results) to fully typed data can require additional strategies. This can introduce both additional complexity and a learning curve for developers used to strict type enforcement. Some of these strategies could be:

1. **Drop Typing for Intermediate Results**
   One option is to simply drop typing for the intermediate results and work with raw `dynamic` or `object` types (e.g., `object` in C# or `Map<String, Object>` in Java). This is not ideal, as it sacrifices type safety during the merging process.

2. **Source Code Generation**
   In languages like **C#**, one powerful approach is to use **source generation** to create a `RecursivePartial`-like type automatically during build time. This allows for type-safe merging of fragments while retaining the flexibility to handle intermediate incomplete structures.

The key takeaway is that the **intermediate result** may not always match the final expected type or schema because it is built incrementally. This requires careful handling of types, especially for languages with static typing.

> [!NOTE]
> It is important to note that the constraint of handling incomplete, intermediate results during the merging process introduces significant complexity, in real-world applications, there are often other alternatives (discussed above).
> Because of these simpler and more robust alternatives, this library is not meant for real-world production use but rather as an exploration of the concept of progressive JSON merging. 
> It demonstrates how you can handle incremental data streams and reconstruct a JSON object in an additive, non-conflicting way. 
> However, in most real-world use cases, alternative approaches (like separate APIs or background processing) are likely to be more effective, scalable, and maintainable.

## Testing

All test cases for the client library are located in the `tests/` directory. Each test file represents a self-contained case and includes an **array of steps**, with each step containing:
* `fragment`: A JSON object fragment to apply.
* `success`: A boolean indicating whether the operation is expected to succeed.
* `currentResult`: The full reconstructed object after applying the fragment (only required when `success` is `true`).

A test runner (to be implemented per language) should:
1. Apply each `fragment` in sequence.
2. Assert that:
   * If `success: true`, applying the fragment does not fail and the new result is deeply equal to `currentResult`.
   * If `success: false`, applying the fragment fails.

A JSON Schema for test files is available at:
[`tests/progressive-json-test-case.schema.json`](tests/progressive-json-test-case.schema.json)