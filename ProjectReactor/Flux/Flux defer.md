# Q: What is `Flux.defer()` in Project Reactor?

## Simple explanation

Same concept is available in Mono also (https://github.com/ReactiveJava-Vivid-Vortex/ReactiveJavaWebFluxProjectReactorConcept-2/blob/b6b0698907ea616821dfa8f342ae504e83f339fc/ProjectReactor/CreateVsExecuteAndDefer.md?plain=1#L170), basically it's talking about starting the calling method and not subscription.

Think of `Flux.defer()` as **"Don't create the Flux now. Create it only when someone subscribes."**

Without `defer`, the Flux is created immediately.

With `defer`, the Flux creation is **postponed** until a subscriber actually asks for the data.

---

## Real-life analogy

Imagine a restaurant.

### Without `defer`

The chef cooks the food **before any customer arrives**.

```
Chef: I'll prepare 5 pizzas now.
(Customer comes later)
Customer gets the already prepared pizza.
```

The food may already be old.

---

### With `defer`

The chef waits until a customer orders.

```
Customer arrives
        ↓
Chef starts cooking
        ↓
Fresh pizza is served
```

Every customer gets freshly prepared food.

---

# Example 1 - Without `defer`

```java
Flux<Integer> numbers = Flux.just(getNumber());

numbers.subscribe(System.out::println);
numbers.subscribe(System.out::println);
```

```java
private static int getNumber() {
    System.out.println("Generating number");
    return new Random().nextInt(100);
}
```

### Output

```
Generating number

23
23
```

Notice something?

`Generating number` printed only **once**.

The value `23` was created when `Flux.just()` was executed.

Both subscribers receive the same value.

---

## Timeline

```
Program starts

getNumber()
     ↓
returns 23

Flux.just(23)

Subscriber 1
     ↓
23

Subscriber 2
     ↓
23
```

---

# Example 2 - With `Flux.defer()`

```java
Flux<Integer> numbers =
        Flux.defer(() -> Flux.just(getNumber()));

numbers.subscribe(System.out::println);
numbers.subscribe(System.out::println);
```

Output could be

```
Generating number
45

Generating number
81
```

Now notice

Each subscriber caused

```
Flux creation
      ↓
getNumber()
      ↓
new random value
```

Every subscription creates a brand-new Flux.

---

## Timeline

```
Subscriber 1
      ↓
Create Flux
      ↓
getNumber()
      ↓
45

-----------------

Subscriber 2
      ↓
Create Flux
      ↓
getNumber()
      ↓
81
```

---

# Why not just use `Flux.just()`?

Suppose your Flux depends on the current time.

Without `defer`

```java
Flux<Long> time =
    Flux.just(System.currentTimeMillis());
```

Wait 10 seconds...

```java
time.subscribe(System.out::println);
```

Output

```
1754500000000
```

Wait another 10 seconds...

```java
time.subscribe(System.out::println);
```

Output

```
1754500000000
```

Same time!

Because it was captured when the Flux was created.

---

Using `defer`

```java
Flux<Long> time =
    Flux.defer(() ->
        Flux.just(System.currentTimeMillis()));
```

Subscriber 1

```
1754500000000
```

Subscriber 2 (10 seconds later)

```
1754500010000
```

Now every subscriber gets the current time.

---

# Common use cases

## 1. Current timestamp

```java
Flux.defer(() ->
    Flux.just(LocalDateTime.now()));
```

Every subscriber gets the latest time.

---

## 2. Random values

```java
Flux.defer(() ->
    Flux.just(new Random().nextInt()));
```

Fresh random value for every subscriber.

---

## 3. Database query

Instead of

```java
Flux<User> users = database.findAll();
```

use

```java
Flux<User> users =
    Flux.defer(() -> database.findAll());
```

Now the database query runs **when someone subscribes**, not when the Flux variable is created.

---

## 4. Reading a file

```java
Flux<String> file =
    Flux.defer(() -> readFile());
```

The file is opened only when needed.

---

## 5. Calling an external API

```java
Flux<Response> response =
    Flux.defer(() -> webClient.get());
```

Each subscriber makes a fresh API call.

---

# When should you use `defer()`?

Use `defer()` whenever creating the Flux involves:

- Getting the current time
- Generating random values
- Reading a file
- Calling a database
- Calling another service/API
- Any operation whose result may change over time
- Any expensive operation that should happen only when someone subscribes

---

# `Flux.just()` vs `Flux.defer()`

| `Flux.just()` | `Flux.defer()` |
|---------------|----------------|
| Creates the Flux immediately | Creates the Flux when subscribed |
| Same data for all subscribers | Fresh data for every subscriber |
| Good for fixed values | Good for dynamic values |
| Eager creation | Lazy creation |

---

# Easy rule to remember

- **`Flux.just()` = "Create it now."**
- **`Flux.defer()` = "Create it later, when someone subscribes."**

---

# Relationship with Cold Publishers

A `Flux.defer()` is still a **cold publisher**.

Why?

Because every subscriber gets **its own newly created Flux**.

```
Subscriber 1
      ↓
Create new Flux
      ↓
Gets fresh data

Subscriber 2
      ↓
Create another new Flux
      ↓
Gets fresh data
```

This matches the behavior of cold publishers, where each subscriber triggers its own data production.

---

# Interview one-liner

> **`Flux.defer()` delays the creation of a Flux until subscription time. It is useful when the data should be generated or fetched fresh for each subscriber, such as current timestamps, random values, database queries, file reads, or external API calls.**
