# Q: What is `Flux.create()` in Project Reactor?

## Simple explanation

Think of `Flux.create()` as **creating your own data source manually**.

Normally, Reactor creates a `Flux` from an existing source:

- Array → `Flux.fromArray()`
- List → `Flux.fromIterable()`
- Range → `Flux.range()`
- Timer → `Flux.interval()`

But what if **there is no existing source** and **you want to push values yourself whenever they become available**?

That's exactly why `Flux.create()` exists.

---

## Real-life analogy

Imagine you're organizing a live event.

- Audience = Subscriber
- Stage microphone = `FluxSink`
- Host = Your code

The host can announce anything at any time.

```
Host: "Person 1 arrived"
Host: "Person 2 arrived"
Host: "Person 3 arrived"
Host: "Event finished"
```

Nobody knows beforehand how many announcements there will be.

That's exactly how `Flux.create()` works.

---

## Syntax

```java
Flux.create(sink -> {
    // push values manually
});
```

Here,

```
sink
```

is a `FluxSink`.

Think of it as the **pipe through which you send events to subscribers.**

---

# Smallest Example

```java
Flux<Integer> numbers = Flux.create(sink -> {

    sink.next(1);
    sink.next(2);
    sink.next(3);

    sink.complete();

});
```

Subscriber

```java
numbers.subscribe(System.out::println);
```

Output

```
1
2
3
```

Notice that **you** decided

- when to send
- what to send
- when to finish

Reactor didn't generate anything.

---

# How does it work internally?

When someone subscribes

```
Subscriber
      │
      ▼
Flux.create()
      │
      ▼
Your code executes
      │
      ▼
sink.next(...)
      │
      ▼
Subscriber receives value
```

---

# `FluxSink`

The sink is the object used to communicate with subscribers.

Main methods are

```java
sink.next(value);
```

Send one item.

```java
sink.complete();
```

Tell subscriber everything is finished.

```java
sink.error(exception);
```

Tell subscriber something went wrong.

Exactly the same three Reactor signals:

```
next
next
next
complete
```

or

```
next
next
error
```

---

# Example 2

Imagine reading sensor values.

```java
Flux<String> sensor = Flux.create(sink -> {

    sink.next("22°C");
    sink.next("23°C");
    sink.next("24°C");

    sink.complete();

});
```

Output

```
22°C
23°C
24°C
```

---

# Example 3 — Dynamic data

Suppose you don't know beforehand how many values will come.

```java
Flux.create(sink -> {

    for(int i = 1; i <= 5; i++) {
        sink.next(i);
    }

    sink.complete();

});
```

Output

```
1
2
3
4
5
```

---

# Example 4 — Events arriving later

Imagine someone clicks a button.

Whenever they click,

```
Button Click
      │
      ▼
sink.next("Clicked")
```

Subscriber immediately receives

```
Clicked
Clicked
Clicked
```

This is why `Flux.create()` is commonly used for **event-driven programming**.

---

# Why not use `Flux.just()`?

`Flux.just()`

```java
Flux.just(1,2,3)
```

Everything is already known.

```
1
2
3
```

No changes possible.

---

`Flux.create()`

```java
Flux.create(sink -> {

    sink.next(1);

    // later...
    sink.next(2);

    // much later...
    sink.next(3);

});
```

Values can be produced **whenever they become available**.

---

# Typical use cases

## 1. Wrapping callback APIs

Suppose an old Java API works like this:

```java
listener.onMessage(message);
```

You can convert it into a Flux:

```java
Flux.create(sink -> {

    listener.onMessage(msg -> sink.next(msg));

});
```

Now subscribers receive every callback reactively.

---

## 2. WebSocket messages

Whenever a new WebSocket message arrives,

```
WebSocket
      │
      ▼
sink.next(message)
```

---

## 3. Mouse clicks

Every click

```
Mouse Click
      │
      ▼
sink.next(click)
```

---

## 4. Kafka/Event Queue

Every new Kafka message

```
Kafka
   │
   ▼
sink.next(record)
```

---

## 5. File watcher

Whenever a file changes

```
File Changed
      │
      ▼
sink.next(file)
```

---

# How is it different from `Flux.generate()`?

This is one of the biggest interview questions.

| `Flux.generate()` | `Flux.create()` |
|-------------------|-----------------|
| Synchronous | Can be synchronous or asynchronous |
| Emits **one** item per callback | Can emit **many** items whenever you want |
| Pull-style generation | Push-style generation |
| Best for generating data | Best for wrapping event/callback APIs |

Example:

`generate()`

```java
sink.next(1);   // ✅

sink.next(2);   // ❌ Not allowed
```

Only one `next()` per invocation.

---

`create()`

```java
sink.next(1);
sink.next(2);
sink.next(3);
sink.next(4);
```

Perfectly valid.

---

# Rule to remember

- **`Flux.just()`** → I already have all the values.
- **`Flux.fromIterable()`** → My values are in a collection.
- **`Flux.range()`** → I want a sequence of numbers.
- **`Flux.interval()`** → I want values based on time.
- **`Flux.generate()`** → I want to generate values one at a time, synchronously.
- **`Flux.create()`** → I want to **push values manually**, often from callbacks, listeners, or other asynchronous event sources.

## Mental model

```
Flux.just()
       │
       └── Fixed values

Flux.generate()
       │
       └── Create one value at a time

Flux.create()
       │
       └── I am the producer.
           I decide:
             • when to emit
             • how many values to emit
             • when to complete
             • when to fail
```

This makes `Flux.create()` the go-to choice for **bridging non-reactive, event-driven APIs (listeners, callbacks, WebSockets, Kafka consumers, file watchers, etc.) into the reactive world**.
