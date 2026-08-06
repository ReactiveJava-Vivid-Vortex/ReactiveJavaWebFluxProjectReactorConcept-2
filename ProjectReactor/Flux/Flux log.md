



# Q: Explain `Flux.log()` in simple terms

Think of `Flux.log()` as a **CCTV camera** for your reactive pipeline.

It **doesn't change the data**.
It simply **prints everything happening inside the Flux** so you can understand how Reactor works.

```java
Flux.just("A", "B", "C")
    .log()
    .subscribe(System.out::println);
```

Output (simplified):

```text
onSubscribe
request(unbounded)
onNext(A)
A
onNext(B)
B
onNext(C)
C
onComplete
```

Notice two things:

- `onNext(A)` is printed by `log()`
- `A` is printed by your `subscribe()`

---

# Why do we need `log()`?

Without `log()`

```java
Flux.just("A", "B", "C")
    .subscribe(System.out::println);
```

Output

```text
A
B
C
```

You only know **what data came out**.

You **don't know**

- when subscription happened
- when demand was requested
- whether stream completed
- whether there was an error
- whether items were cancelled

`log()` shows all of that.

---

# What does it log?

## 1. onSubscribe

Means

> "A subscriber has connected."

```text
onSubscribe(...)
```

Example

```java
Flux.just(1,2,3)
    .log()
    .subscribe();
```

Output

```text
onSubscribe
```

Think

```
Restaurant opens.

Customer enters.

onSubscribe
```

---

## 2. request(n)

This is one of the most important Reactor concepts.

It means

> "Subscriber is asking for data."

Example

```text
request(unbounded)
```

or

```text
request(1)
```

Think

Restaurant

Customer says

```
Bring me all dishes.

request(unbounded)
```

or

```
Bring only one dish.

request(1)
```

This is **backpressure** in action.

---

## 3. onNext(value)

Means

> "Here is one item."

Example

```text
onNext(A)
onNext(B)
onNext(C)
```

Equivalent to

```
Take this.

Take this.

Take this.
```

---

## 4. onComplete

Means

> "No more items."

Output

```text
onComplete
```

Restaurant analogy

```
Kitchen says

Finished.
No more food.
```

---

## 5. onError

If something fails

```java
Flux.just(1,2)
    .concatWith(Mono.error(new RuntimeException("Boom")))
    .log()
    .subscribe();
```

Output

```text
onNext(1)
onNext(2)
onError(java.lang.RuntimeException: Boom)
```

Meaning

```
Item
Item
Oops!
```

---

## 6. cancel

If subscriber cancels

```text
cancel()
```

Meaning

```
Customer leaves early.

Stop sending items.
```

---

# Complete Example

```java
Flux.just("Apple", "Banana", "Orange")
    .log()
    .subscribe(System.out::println);
```

Output (simplified)

```text
onSubscribe
request(unbounded)

onNext(Apple)
Apple

onNext(Banana)
Banana

onNext(Orange)
Orange

onComplete
```

Timeline

```
Subscriber joins
      │
      ▼
onSubscribe
      │
      ▼
request(unbounded)
      │
      ▼
onNext(Apple)
      │
      ▼
onNext(Banana)
      │
      ▼
onNext(Orange)
      │
      ▼
onComplete
```

---

# Where should `log()` be placed?

Since every operator creates a new `Flux`, placing `log()` at different locations lets you observe different stages of the pipeline.

```java
Flux.just(1,2,3)
    .log("SOURCE")
    .map(i -> i * 10)
    .log("AFTER_MAP")
    .subscribe();
```

Output (simplified)

```text
SOURCE onNext(1)
AFTER_MAP onNext(10)

SOURCE onNext(2)
AFTER_MAP onNext(20)

SOURCE onNext(3)
AFTER_MAP onNext(30)
```

You can clearly see:

```
Source emits 1
        │
        ▼
map()
        │
        ▼
10 emitted
```

This is extremely useful for debugging complex pipelines.

---

# Adding a custom name

Instead of the default operator name:

```java
Flux.just(1,2,3)
    .log("Orders")
    .subscribe();
```

Output

```text
Orders | onSubscribe
Orders | request(unbounded)
Orders | onNext(1)
Orders | onComplete
```

This makes logs much easier to read when you have multiple streams.

---

# `log()` vs `doOnNext()`

These are often confused.

### `log()`

Logs **everything** related to the Reactive Streams lifecycle.

```java
Flux.just(1,2)
    .log()
```

Shows

- onSubscribe
- request
- onNext
- onComplete
- onError
- cancel

---

### `doOnNext()`

Runs your code **only when an item is emitted**.

```java
Flux.just(1,2)
    .doOnNext(System.out::println)
```

Output

```text
1
2
```

No subscription information.
No request information.
No completion information.

---

# When should you use `log()`?

Use it when you want to debug:

- Why is my Flux not emitting?
- Is someone subscribing?
- Is backpressure working?
- Is the stream completing?
- Is it getting cancelled?
- Where is an error occurring?

It is mainly a **debugging tool**, not something you typically leave enabled in production because it can generate a large amount of log output.

---

# One-line summary

`Flux.log()` is a **debugging operator** that logs the entire Reactive Streams lifecycle—**subscription (`onSubscribe`), demand (`request`), emitted items (`onNext`), completion (`onComplete`), errors (`onError`), and cancellation (`cancel`)—without modifying the data flowing through the stream.**

---

# Q: How is `log()` different from the `doOn...()` operators in Project Reactor?

The easiest way to remember it is:

| `log()`                                        | `doOn...()`                                           |
| ---------------------------------------------- | ----------------------------------------------------- |
| Built-in debugging operator                    | Side-effect operators                                 |
| Automatically logs all Reactive Streams events | Lets **you** execute your own code on specific events |
| Used mainly for debugging                      | Used for logging, metrics, auditing, cleanup, etc.    |
| No customization of the log message            | Full control over what happens                        |

---

# Think of it like CCTV vs Security Guard

Imagine a restaurant.

## `log()` = CCTV Camera 📹

The CCTV automatically records everything.

* Customer enters
* Customer orders
* Food served
* Customer leaves

You don't write any code.

```java
Flux.just("Pizza", "Burger")
    .log()
    .subscribe();
```

Output

```text
onSubscribe
request(unbounded)
onNext(Pizza)
onNext(Burger)
onComplete
```

The CCTV records every important event automatically.

---

## `doOn...()` = Security Guard 👮

The security guard only acts when you tell him.

Example

```java
Flux.just("Pizza", "Burger")
    .doOnNext(item -> System.out.println("Serving " + item))
    .subscribe();
```

Output

```text
Serving Pizza
Serving Burger
```

Here **you decided** what should happen.

---

# `log()` logs everything

```java
Flux.just(1,2,3)
    .log()
    .subscribe();
```

Logs

```text
onSubscribe
request(unbounded)
onNext(1)
onNext(2)
onNext(3)
onComplete
```

Notice that you never wrote any logging code.

---

# `doOn...()` lets you choose

Instead of logging everything, you choose the event.

```java
Flux.just(1,2,3)
    .doOnSubscribe(s -> System.out.println("Subscribed"))
    .doOnNext(i -> System.out.println("Received " + i))
    .doOnComplete(() -> System.out.println("Done"))
    .subscribe();
```

Output

```text
Subscribed
Received 1
Received 2
Received 3
Done
```

You control the message.

---

# `log()` is equivalent to many `doOn...()` operators

Internally, `log()` is conceptually similar to attaching listeners for every important signal.

```java
Flux.just(1,2,3)
    .doOnSubscribe(...)
    .doOnRequest(...)
    .doOnNext(...)
    .doOnError(...)
    .doOnComplete(...)
    .doOnCancel(...)
```

except Reactor writes the logging code for you.

So you can think of

```java
.log()
```

as a convenient shortcut for

> "Log every Reactive Streams signal."

---

# Example 1: Using `log()`

```java
Flux.just("A", "B")
    .log()
    .subscribe();
```

Output

```text
onSubscribe
request(unbounded)
onNext(A)
onNext(B)
onComplete
```

---

# Example 2: Using `doOnNext()`

```java
Flux.just("A", "B")
    .doOnNext(s -> System.out.println("Saving " + s))
    .subscribe();
```

Output

```text
Saving A
Saving B
```

No subscription logs.
No request logs.
No completion logs.

---

# Example 3: Combining both

Very common in real applications.

```java
Flux.just("A", "B")
    .log()
    .doOnNext(item -> System.out.println("Updating DB: " + item))
    .subscribe();
```

Output

```text
onSubscribe
request(unbounded)
onNext(A)
Updating DB: A
onNext(B)
Updating DB: B
onComplete
```

Here:

* `log()` shows what Reactor is doing.
* `doOnNext()` performs your custom action.

---

# Common `doOn...()` operators

| Operator          | Triggered when                                   | Typical use                            |
| ----------------- | ------------------------------------------------ | -------------------------------------- |
| `doOnSubscribe()` | Subscriber subscribes                            | Log subscription, initialize resources |
| `doOnRequest()`   | Subscriber requests items                        | Observe backpressure                   |
| `doOnNext()`      | Item is emitted                                  | Logging, metrics, auditing             |
| `doOnError()`     | Error occurs                                     | Log errors, send alerts                |
| `doOnComplete()`  | Flux completes successfully                      | Success logging                        |
| `doOnTerminate()` | Complete or error                                | Common cleanup                         |
| `doFinally()`     | Complete, error, or cancel                       | Always release resources               |
| `doOnCancel()`    | Subscription is cancelled                        | Cleanup or logging                     |
| `doOnEach()`      | Every signal (`onNext`, `onError`, `onComplete`) | Advanced signal inspection             |

---

# When should you use which?

## Use `log()` when...

* You are debugging a reactive pipeline.
* You want to understand how Reactor behaves.
* You want to see requests, subscriptions, completions, and cancellations automatically.

Example:

```java
Flux.just(1,2,3)
    .log();
```

---

## Use `doOn...()` when...

* You want to perform side effects.
* You want custom log messages.
* You want to update metrics.
* You want to write audit records.
* You want cleanup logic.

Example:

```java
Flux.just(order)
    .doOnNext(orderService::saveAuditLog)
    .doOnError(alertService::notifyAdmin)
    .doFinally(signal -> connection.close());
```

---

# Can `doOn...()` modify the data?

No.

Just like `log()`, the `doOn...()` operators are **side-effect operators**. They observe the stream but do **not** change the items.

```java
Flux.just(1,2,3)
    .doOnNext(i -> System.out.println(i))
    .map(i -> i * 10)
```

The `doOnNext()` prints `1`, `2`, `3`, but the values continue unchanged into `map()`, which transforms them to `10`, `20`, `30`.

---

# One-line summary

* **`log()`** is Reactor's built-in **automatic debugger** that logs the entire Reactive Streams lifecycle.
* **`doOn...()`** operators are **custom side-effect hooks** that let you execute your own code when specific lifecycle events (subscribe, next, error, complete, cancel, etc.) occur, without modifying the data.

---

# Q: Is the `log()` operator also available for `Mono`?

Yes. **`Mono` also has the `log()` operator**, and it behaves exactly like `Flux.log()`.

The only difference is that a `Mono` can emit **at most one item**, while a `Flux` can emit many.

---

# Example 1: `Mono` emitting a value

```java
Mono.just("Hello")
    .log()
    .subscribe(System.out::println);
```

Output (simplified)

```text
onSubscribe
request(unbounded)
onNext(Hello)
Hello
onComplete
```

Notice that there is only one `onNext()` because a `Mono` can emit only one value.

---

# Example 2: Empty `Mono`

```java
Mono.empty()
    .log()
    .subscribe();
```

Output

```text
onSubscribe
request(unbounded)
onComplete
```

There is **no `onNext()`** because no value is emitted.

---

# Example 3: Error `Mono`

```java
Mono.error(new RuntimeException("Boom"))
    .log()
    .subscribe();
```

Output

```text
onSubscribe
request(unbounded)
onError(java.lang.RuntimeException: Boom)
```

Again, there is **no `onNext()`** because an error occurs before any value is emitted.

---

# Comparison: `Mono.log()` vs `Flux.log()`

| Feature             | `Mono.log()` | `Flux.log()` |
| ------------------- | ------------ | ------------ |
| Available?          | ✅ Yes        | ✅ Yes        |
| Logs subscription   | ✅            | ✅            |
| Logs request        | ✅            | ✅            |
| Logs emitted values | Maximum 1    | 0 to N       |
| Logs completion     | ✅            | ✅            |
| Logs errors         | ✅            | ✅            |
| Logs cancellation   | ✅            | ✅            |

---

# Why do both have `log()`?

Both `Mono` and `Flux` implement the **Reactive Streams** specification. They both go through the same lifecycle:

```
onSubscribe
      ↓
request
      ↓
onNext (optional)
      ↓
onComplete OR onError
```

The only difference is:

* **`Mono`** → emits **0 or 1** item.
* **`Flux`** → emits **0 to many** items.

Since the lifecycle is the same, Reactor provides the same debugging operator (`log()`) for both.

---

# One-line summary

**Yes. `Mono.log()` works exactly like `Flux.log()`, logging the entire Reactive Streams lifecycle (`onSubscribe`, `request`, `onNext`, `onComplete`, `onError`, and `cancel`). The only practical difference is that a `Mono` can produce at most one `onNext` event, whereas a `Flux` can produce many.**
