



### Q: Explain `FluxSink` and its thread safety in simple terms.

Think of **`FluxSink` as a microphone** connected to a `Flux`.

- The **Flux** is the audience listening.
- The **FluxSink** is the microphone through which you send messages to the audience.

You don't create a `FluxSink` directly. Reactor gives it to you inside `Flux.create()` or `Flux.push()`.

```java
Flux<String> flux = Flux.create(sink -> {
    sink.next("A");
    sink.next("B");
    sink.complete();
});
```

Here:

- `sink.next(...)` → send a value (`onNext`)
- `sink.complete()` → tell subscribers there are no more values (`onComplete`)
- `sink.error(...)` → end with an error (`onError`)

---

# Why do we need FluxSink?

Normally, a `Flux` already knows where its data comes from.

Example:

```java
Flux.just("A", "B", "C");
```

But sometimes **your data comes from outside Reactor**.

Examples:

- Button clicks
- Kafka listener
- RabbitMQ consumer
- TCP socket
- Legacy callback API
- File watcher

When an external event happens, you need some way to "push" that event into Reactor.

That's exactly what `FluxSink` is for.

```
External Event
      │
      ▼
 FluxSink.next()
      │
      ▼
     Flux
      │
      ▼
 Subscriber
```

---

# Simple Example

Imagine someone rings your doorbell.

Every time the bell rings:

```
Bell rings
    │
    ▼
sink.next("Someone is here")
    │
    ▼
Subscriber receives it
```

Code:

```java
Flux<String> doorBell = Flux.create(sink -> {

    sink.next("Person 1");
    sink.next("Person 2");

    sink.complete();
});
```

Output

```
Person 1
Person 2
Completed
```

---

# What exactly is FluxSink?

It is simply an object that lets you emit Reactor signals.

```
sink.next(value)
```

creates

```
onNext(value)
```

```
sink.complete()
```

creates

```
onComplete()
```

```
sink.error(ex)
```

creates

```
onError(ex)
```

So `FluxSink` is basically a **signal producer**.

---

# Where does FluxSink come from?

Inside

```java
Flux.create(...)
```

```java
Flux.create(sink -> {

});
```

Reactor internally creates the sink.

You simply use it.

---

# Real-world Example

Suppose an old library gives callbacks.

```java
legacyApi.registerListener(data -> {
    // callback
});
```

To convert this into Reactor:

```java
Flux<String> flux = Flux.create(sink -> {

    legacyApi.registerListener(data -> {
        sink.next(data);
    });

});
```

Now every callback becomes a Flux event.

```
Legacy API
      │
      ▼
Callback
      │
      ▼
sink.next(data)
      │
      ▼
Flux
      │
      ▼
Subscriber
```

This is one of the biggest uses of `Flux.create()` and `FluxSink`.

---

# Thread Safety

This is where many people get confused.

Imagine two people trying to write on the same whiteboard.

```
Person A
           \
            \
             Whiteboard
            /
           /
Person B
```

If both write simultaneously:

```
Hello
      WorHelld
```

Everything becomes mixed.

The same thing can happen with `FluxSink`.

---

## Case 1 — One thread (Safe)

```
Thread-1

sink.next(1)

sink.next(2)

sink.next(3)
```

Output

```
1
2
3
```

No problem.

---

## Case 2 — Multiple threads

Imagine:

```
Thread A

sink.next("A")
```

At the exact same time

```
Thread B

sink.next("B")
```

Now both threads are modifying the same stream simultaneously.

Without proper coordination, you could get issues such as:
- unexpected ordering,
- race conditions,
- or concurrent emission problems.

---

# Does Reactor make FluxSink thread-safe?

**Yes—but it depends on how you create the Flux.**

## `Flux.create()`

`Flux.create()` is designed to support **multiple producer threads**. Reactor serializes the signals so subscribers see them in a valid sequence.

Example:

```java
Flux<Integer> flux = Flux.create(sink -> {

    new Thread(() -> sink.next(1)).start();

    new Thread(() -> sink.next(2)).start();

});
```

Both threads can emit safely.

However, the **order is not guaranteed** because the threads race.

Possible outputs:

```
1
2
```

or

```
2
1
```

Both are correct.

---

## `Flux.push()`

`Flux.push()` is optimized for **a single producer thread**.

Example:

```java
Flux.push(sink -> {
    sink.next(1);
    sink.next(2);
});
```

Fast and efficient.

But **don't call `sink.next()` concurrently from multiple threads** with `Flux.push()`. That usage is not supported.

---

# Why have both?

### `Flux.create()`

Supports:

```
Producer A
Producer B
Producer C

      ↓
    FluxSink
```

Multiple threads.

---

### `Flux.push()`

Supports:

```
Producer

   ↓

FluxSink
```

One thread only.

Because Reactor doesn't need to coordinate multiple threads, `Flux.push()` can be a little more efficient.

---

# Which one should you use?

| Situation | Use |
|-----------|-----|
| Events can come from multiple threads | `Flux.create()` |
| Only one thread produces events | `Flux.push()` |
| Wrapping callback APIs where callbacks may come from different threads | `Flux.create()` |
| Wrapping a single-threaded event source | `Flux.push()` |

---

# Easy way to remember

Think of a microphone:

- **Flux** = the audience.
- **FluxSink** = the microphone.
- `sink.next()` = speak one message.
- `sink.complete()` = say "That's all."
- `sink.error()` = announce something went wrong and stop.

For thread safety:

- **`Flux.create()`** = a microphone with a coordinator, so multiple speakers can use it safely (though whoever speaks first is not deterministic).
- **`Flux.push()`** = a microphone intended for **one speaker only**. If multiple people try to speak into it at the same time, it's not a supported use case.

### Interview takeaway

> `FluxSink` is the object used to imperatively emit signals (`onNext`, `onComplete`, and `onError`) into a `Flux`, typically when adapting callback-based or event-driven APIs to Reactor. `Flux.create()` supports concurrent emissions from multiple producer threads, whereas `Flux.push()` is intended for a single producer thread.
