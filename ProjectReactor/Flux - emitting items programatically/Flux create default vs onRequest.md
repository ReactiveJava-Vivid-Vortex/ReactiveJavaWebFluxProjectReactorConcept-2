### Q: Explain `Flux.create()` default behavior and `onRequest` (on demand) in simple terms.

The easiest way to understand this is to first understand **backpressure**.

Imagine you own a pizza shop.

* **Customer** = Subscriber
* **Chef** = `Flux.create()`
* **Pizza** = Data
* **Order count** = Demand (`request(n)`)

The customer may say:

> "Please give me **2 pizzas first**."

That means:

```
request(2)
```

The chef should ideally make only 2 pizzas.

---

# 1. Default behavior of `Flux.create()`

By default, `Flux.create()` is **push-based**.

That means the producer pushes data whenever it wants, **without waiting for the subscriber to ask**.

Example:

```java
Flux<Integer> flux = Flux.create(sink -> {

    for (int i = 1; i <= 5; i++) {
        sink.next(i);
    }

    sink.complete();
});
```

Subscriber:

```java
flux.subscribe(System.out::println);
```

Output

```
1
2
3
4
5
```

Notice something:

The producer never asked

> "How many items do you want?"

It simply pushed everything.

Think of it like this:

```
Producer

1
2
3
4
5

↓↓↓↓↓

Subscriber
```

The producer is in full control.

---

# What if the subscriber only wants 2 items?

Imagine the subscriber says:

```
Give me only 2.
```

But the producer already produced

```
1
2
3
4
5
```

Now Reactor has to decide what to do with the extra items.

That's where **backpressure strategies** (BUFFER, DROP, LATEST, ERROR, IGNORE) come into play. These define how to handle items produced faster than they're requested.

---

# 2. What is `onRequest()`?

Sometimes you don't want to produce everything immediately.

Instead, you want to produce **only when someone asks**.

This is called **on-demand production**.

`FluxSink` provides:

```java
sink.onRequest(...)
```

This callback runs whenever the subscriber requests more items.

---

Example:

```java
Flux<Integer> flux = Flux.create(sink -> {

    sink.onRequest(n -> {

        System.out.println("Requested: " + n);

        for (int i = 1; i <= n; i++) {
            sink.next(i);
        }
    });

});
```

Suppose the subscriber requests:

```
request(3)
```

Then:

```
Requested: 3

1
2
3
```

Nothing is produced before the request.

---

# Pizza Shop Example

## Default (`Flux.create()`)

Customer walks in.

Chef immediately cooks

```
🍕🍕🍕🍕🍕🍕🍕🍕🍕🍕
```

Customer says

> "I only wanted 2."

The remaining pizzas are extra.

---

## `onRequest()`

Customer says

> "Give me 2 pizzas."

Chef cooks only

```
🍕🍕
```

Later customer says

> "Give me 3 more."

Chef cooks

```
🍕🍕🍕
```

No waste.

---

# Flow Diagram

### Default behavior

```
Flux.create()

Producer
   │
   │ push immediately
   ▼
Flux
   │
   ▼
Subscriber
```

The producer decides **when** and **how much** to emit.

---

### `onRequest()`

```
Subscriber

request(3)

     │
     ▼

FluxSink.onRequest()

     │
     ▼

Producer emits 3 items
```

The subscriber drives the flow.

---

# Real-world Example

Suppose you're reading a huge file with 10 million lines.

### Default

```java
Flux.create(sink -> {

    while(file.hasNext()) {
        sink.next(file.readLine());
    }

});
```

The producer starts reading the whole file immediately, even if the subscriber only needs a few lines.

---

### `onRequest()`

```java
Flux.create(sink -> {

    sink.onRequest(n -> {

        for (long i = 0; i < n; i++) {
            sink.next(file.readLine());
        }

    });

});
```

Now if the subscriber requests:

```
request(10)
```

Only **10 lines** are read.

Later:

```
request(20)
```

Only **20 more lines** are read.

This is much more efficient for large or expensive data sources.

---

# Does `onRequest()` happen automatically?

No.

If you don't register an `onRequest()` callback:

```java
Flux.create(sink -> {

    sink.next(1);
    sink.next(2);
    sink.next(3);

});
```

Reactor does **not** stop you from emitting. You are simply pushing data as you choose.

If you do register `onRequest()`:

```java
sink.onRequest(n -> {
    // produce data based on demand
});
```

then you can make your producer respect the subscriber's demand.

---

# Default vs. `onRequest()`

| Feature                | Default `Flux.create()`                                  | `Flux.create()` with `onRequest()`                                          |
| ---------------------- | -------------------------------------------------------- | --------------------------------------------------------------------------- |
| Who controls emission? | Producer                                                 | Subscriber (through demand)                                                 |
| Data production        | Immediately                                              | Only when requested                                                         |
| Suitable for           | Event sources (UI events, Kafka callbacks, sockets)      | Pull-style or expensive data sources (files, databases, paginated APIs)     |
| Backpressure awareness | Not by itself; you choose an overflow strategy if needed | You can naturally honor backpressure by producing only the requested amount |

### Easy way to remember

* **Default `Flux.create()`** = A waiter who keeps placing dishes on your table without asking if you're ready.
* **`Flux.create()` + `onRequest()`** = A waiter who asks, **"How many dishes would you like now?"**, and serves only that many.

That's why `onRequest()` is commonly used when adapting data sources that can produce data **on demand**, while the default push behavior is ideal for event sources where events occur whenever they occur (mouse clicks, Kafka messages, WebSocket messages, etc.).
