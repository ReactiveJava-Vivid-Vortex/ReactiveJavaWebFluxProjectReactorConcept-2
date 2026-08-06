> **Are there only three types of signals in Project Reactor (and therefore Spring WebFlux, since it is based on Reactor): `onNext`, `onComplete`, and `onError`? Please explain them in simple terms.**

## Q1. Is it correct that there are only three types of signals?

<mark> Signlas are the only medium of commuication in Reactive programming</mark>

**Yes, with one small addition.**

Every `Mono` and `Flux` communicate with their subscribers using **signals**. Think of signals as **messages** sent from the publisher to the subscriber.

There are actually **4 signals** involved in a reactive stream:

| Signal | Who sends it? | Meaning |
|---------|---------------|---------|
| `onSubscribe()` | Publisher | "Connection established. We're ready." |
| `onNext(T)` | Publisher | "Here is a value." |
| `onComplete()` | Publisher | "I'm done successfully." |
| `onError(Throwable)` | Publisher | "Something went wrong." |

However, when people talk about **data flow**, they usually focus on these **three terminal/data signals**:

- `onNext`
- `onComplete`
- `onError`

So your understanding is **essentially correct**.

---

# Think of it like ordering food

Imagine you ordered pizza.

### Signal 1: onSubscribe

Restaurant says:

> "We've received your order."

Nothing has been delivered yet.

```
onSubscribe()
```

---

### Signal 2: onNext

Delivery boy arrives.

> "Here's your pizza."

```
onNext(Pizza)
```

For a `Flux`, this can happen many times.

```
onNext(Pizza1)
onNext(Pizza2)
onNext(Pizza3)
```

For a `Mono`, it can happen **at most once**.

---

### Signal 3: onComplete

Restaurant says:

> "That's all. No more pizzas."

```
onComplete()
```

This means everything finished successfully.

---

### Signal 4: onError

Instead, suppose the restaurant calls:

> "Sorry, we burned your pizza."

```
onError(Exception)
```

The stream immediately ends.

No more pizzas.
No completion signal.

---

# Important Rule

A stream always ends in **exactly one** of these two terminal signals:

```
onComplete()
```

OR

```
onError()
```

Never both.

---

## Example 1

```java
Flux.just(1, 2, 3)
```

Signals are:

```
onSubscribe()

onNext(1)

onNext(2)

onNext(3)

onComplete()
```

---

## Example 2

```java
Flux.error(new RuntimeException("Oops"))
```

Signals:

```
onSubscribe()

onError(RuntimeException)
```

Notice there is:

- no `onNext`
- no `onComplete`

---

## Example 3

```java
Flux.just(1, 2)
        .concatWith(Flux.error(new RuntimeException()));
```

Signals:

```
onSubscribe()

onNext(1)

onNext(2)

onError(RuntimeException)
```

Again,

No `onComplete()`.

---

# Can `onComplete()` and `onError()` both happen?

**No.**

These are called **terminal signals**.

Once one happens, the stream is over.

```
onNext()

onNext()

onComplete()

❌ onNext() again
```

Not allowed.

Similarly,

```
onNext()

onError()

❌ onComplete()
```

Not allowed.

---

# What about an empty Flux?

```java
Flux.empty()
```

Signals:

```
onSubscribe()

onComplete()
```

No data was emitted.

---

# What about Mono?

### Success

```java
Mono.just("Hello")
```

Signals

```
onSubscribe()

onNext("Hello")

onComplete()
```

---

### Empty

```java
Mono.empty()
```

Signals

```
onSubscribe()

onComplete()
```

---

### Error

```java
Mono.error(new RuntimeException())
```

Signals

```
onSubscribe()

onError(RuntimeException)
```

---

# A simple way to remember

Think of every Reactor stream like a conversation between the publisher and subscriber:

```
Publisher:
"Ready?"
↓
onSubscribe()

Publisher:
"Here's some data."
↓
onNext()

Publisher:
"Here's more data."
↓
onNext()

Publisher:
"I'm finished."
↓
onComplete()
```

or

```
Publisher:
"Ready?"
↓
onSubscribe()

Publisher:
"Here's some data."
↓
onNext()

Publisher:
"Oops! Something went wrong."
↓
onError()
```

---

# Why is this important?

Almost every Reactor operator reacts to one or more of these signals:

| Signal | Common operators |
|---------|------------------|
| `onNext` | `map()`, `filter()`, `flatMap()`, `doOnNext()` |
| `onComplete` | `then()`, `doOnComplete()` (Flux), `thenMany()` |
| `onError` | `onErrorResume()`, `onErrorReturn()`, `doOnError()` |
| All signals | `log()`, `doFinally()`, `materialize()` |

Once you understand these signals, you'll find it much easier to reason about how operators behave.

### Rule to remember

A Reactor stream follows this lifecycle:

```
onSubscribe()

        ↓

Zero or more onNext()

        ↓

Exactly one terminal signal

        ↓
   onComplete()
        OR
   onError()
```

This simple lifecycle explains the behavior of virtually every `Mono` and `Flux` you will encounter in Spring WebFlux.

---

Your question is good. A slightly more natural version would be:

> **How do these signals make reactive programming different from traditional (non-reactive) Java?**

## Q1. How do these signals make reactive programming different from traditional Java?

The biggest difference is this:

* **Traditional Java:** Your code **pulls** data when it needs it.
* **Reactive (Project Reactor):** The publisher **pushes** signals to the subscriber.

Think of it like this:

### Traditional Java (Pull Model)

Imagine you're reading a book.

You decide when to turn each page.

```text
You → Give me page 1
Book → Page 1

You → Give me page 2
Book → Page 2

You → Give me page 3
Book → Page 3
```

The **caller controls everything**.

Example:

```java
List<Integer> numbers = List.of(1, 2, 3);

for (Integer n : numbers) {
    System.out.println(n);
}
```

Here:

* You ask for each element.
* There is **no concept of signals** like `onNext()` or `onComplete()`.
* The method simply returns the data.

---

## Reactive Java (Push Model)

Now imagine subscribing to a YouTube channel.

You don't keep asking:

> "Any new video?"
>
> "Any new video?"
>
> "Any new video?"

Instead:

```text
You subscribe

↓

Whenever a new video is uploaded

↓

You receive a notification
```

That's exactly how Reactor works.

The publisher sends signals whenever it has something to communicate.

```text
Publisher

↓

onSubscribe()

↓

onNext(1)

↓

onNext(2)

↓

onNext(3)

↓

onComplete()
```

Notice:

The subscriber never asks for the next value explicitly. It simply reacts to incoming signals.

---

# Traditional Java returns data

A normal Java method looks like this:

```java
public List<String> getNames() {
    return List.of("John", "Alice", "Bob");
}
```

Execution:

```text
Call method

↓

Method executes completely

↓

Returns List

↓

Method is finished
```

Everything happens synchronously.

---

# Reactor returns a pipeline

```java
Flux<String> names = Flux.just("John", "Alice", "Bob");
```

Nothing has happened yet.

The `Flux` is just a **recipe** describing how values will be produced.

Only when someone subscribes:

```java
names.subscribe(System.out::println);
```

do the signals start flowing:

```text
onSubscribe()

↓

onNext("John")

↓

onNext("Alice")

↓

onNext("Bob")

↓

onComplete()
```

This is why Reactor is often described as **lazy**.

---

# Errors are also signals

Traditional Java uses exceptions:

```java
try {
    int result = divide(a, b);
} catch (Exception e) {
    // Handle error
}
```

The exception interrupts the normal flow immediately.

---

Reactive treats an error as another signal:

```text
onNext()

↓

onNext()

↓

onError()
```

Operators like `onErrorResume()` can react to that signal and continue with a fallback stream.

---

# Completion is also explicit

Traditional Java doesn't tell you:

> "I'm done."

When a method returns, you simply know it's finished.

Reactive makes completion an explicit signal:

```text
onComplete()
```

Many operators (`then()`, `doOnComplete()`, etc.) rely on this signal.

---

# Why is this useful?

Consider reading a huge log file.

### Traditional Java

```java
List<String> lines = Files.readAllLines(path);
```

The entire file is read into memory before you can process the first line.

```text
Read file

↓

Create List

↓

Return List

↓

Start processing
```

---

### Reactive

```java
Flux<String> lines = readFile();
```

Signals might look like:

```text
onNext(line1)

↓

Process line1

↓

onNext(line2)

↓

Process line2

↓

...

↓

onComplete()
```

You can start processing immediately without waiting for the entire file to be read.

---

# Event-driven instead of call-driven

Traditional Java is typically **call-driven**:

```text
Call method

↓

Get result

↓

Continue
```

Reactive is **event-driven**:

```text
Subscribe

↓

Wait for signals

↓

React to each signal

↓

Finish
```

The focus shifts from "calling methods" to "responding to events."

---

# Simple comparison

| Traditional Java             | Project Reactor                              |
| ---------------------------- | -------------------------------------------- |
| Pulls data                   | Receives pushed signals                      |
| Method returns value         | Publisher emits signals                      |
| Exceptions are thrown        | Errors are emitted as `onError()`            |
| Method return means finished | `onComplete()` explicitly signals completion |
| Usually synchronous          | Asynchronous and non-blocking by design      |
| Eager execution              | Lazy execution (starts on `subscribe()`)     |

## The most important insight

The real power of Reactor is **not just that it's asynchronous**. It's that **everything—data, completion, and errors—is represented as a stream of signals**.

Instead of thinking:

> "Call a method and get a value."

You start thinking:

> "Subscribe to a stream and react to the signals it emits."

That shift—from **return values** to **signal-driven communication**—is the fundamental difference between traditional Java and Project Reactor.
