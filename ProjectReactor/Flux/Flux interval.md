



## Q: Explain `Flux.interval()` in simple terms

Think of `Flux.interval()` as a **timer that keeps producing numbers at fixed time intervals**.

Instead of emitting all values immediately like `Flux.range()`, it **waits for a specified duration before emitting each value**.

### Real-life analogy

Imagine a clock that rings **every second**.

```
1 second  -> Bell rings (0)
2 seconds -> Bell rings (1)
3 seconds -> Bell rings (2)
4 seconds -> Bell rings (3)
...
```

That's exactly what `Flux.interval()` does.

---

## Simplest Example

```java
Flux.interval(Duration.ofSeconds(1))
    .subscribe(System.out::println);

Thread.sleep(5000);
```

### Output

```
(after 1 second)
0

(after 2 seconds)
1

(after 3 seconds)
2

(after 4 seconds)
3
```

Notice:

- It starts from **0**
- Produces **Long** values
- Keeps running forever until cancelled

---

## What is actually happening?

Imagine Reactor has an internal timer.

```
Time

0 sec   1 sec   2 sec   3 sec   4 sec

 |--------|--------|--------|--------|

          0        1        2        3
```

Every time the timer expires, Reactor emits the next number.

---

## Syntax

```java
Flux.interval(Duration.ofSeconds(1))
```

means

> "Emit one item every one second."

---

## Another Example

Every 500 milliseconds

```java
Flux.interval(Duration.ofMillis(500))
    .subscribe(System.out::println);
```

Output

```
0
1
2
3
4
5
...
```

Each number is half a second apart.

---

# Is it lazy?

Yes.

Like almost every Reactor publisher, **`Flux.interval()` is lazy**.

Nothing starts until someone subscribes.

```java
Flux<Long> flux = Flux.interval(Duration.ofSeconds(1));

System.out.println("Created");
```

Output

```
Created
```

Nothing happens.

After subscribing:

```java
flux.subscribe(System.out::println);
```

Now the timer starts.

---

# Infinite Publisher

Unlike `Flux.range()`,

```java
Flux.range(1,5)
```

which emits

```
1
2
3
4
5
Complete
```

`Flux.interval()` never finishes by itself.

```
0
1
2
3
4
5
6
7
8
...
```

It continues forever until:

- cancelled
- disposed
- application stops
- an operator like `take()` limits it

---

## Making it finite

```java
Flux.interval(Duration.ofSeconds(1))
    .take(5)
    .subscribe(System.out::println);
```

Output

```
(after 1 sec)
0

(after 2 sec)
1

(after 3 sec)
2

(after 4 sec)
3

(after 5 sec)
4

Completed
```

---

# Thread used

Unlike `Flux.just()` or `Flux.range()`, `Flux.interval()` does **not** emit on the calling thread.

It uses Reactor's **parallel scheduler** internally.

```java
Flux.interval(Duration.ofSeconds(1))
    .doOnNext(i ->
        System.out.println(Thread.currentThread().getName()))
    .subscribe();
```

Possible output

```
parallel-1
parallel-1
parallel-1
...
```

So it emits asynchronously.

---

# Comparison with `Flux.range()`

| `Flux.range()` | `Flux.interval()` |
|---------------|-------------------|
| Emits immediately | Emits after a delay |
| Synchronous | Asynchronous |
| Finite | Infinite by default |
| Numbers are supplied by you | Numbers are generated automatically |
| Runs on caller thread | Uses Reactor scheduler |

---

# Common Uses

### 1. Polling

Check a database every 10 seconds.

```java
Flux.interval(Duration.ofSeconds(10))
```

---

### 2. Refresh dashboard

Refresh statistics every second.

---

### 3. Heartbeat

Send a heartbeat message every 5 seconds.

---

### 4. Retry periodically

Attempt a connection every few seconds.

---

### 5. Scheduled background work

Run cleanup every minute.

---

# Key points to remember

- Produces an **infinite stream** of `Long` values starting from **0**.
- Emits values **at fixed time intervals**.
- **Lazy**—the timer starts only after `subscribe()`.
- **Asynchronous**—uses Reactor's scheduler instead of the caller thread.
- Never completes unless cancelled or limited (for example, with `take()`).
- Commonly used for timers, polling, heartbeats, and periodic tasks.

---

## Quick Mental Model

```
Flux.just()

Subscribe
    ↓
0 1 2 3 4 (immediately)
Complete


Flux.range()

Subscribe
    ↓
1 2 3 4 5 (immediately)
Complete


Flux.interval()

Subscribe
    ↓
(wait)
0
(wait)
1
(wait)
2
(wait)
3
...
(continues until cancelled)
```

The simplest way to think about `Flux.interval()` is:

> **`Flux.just()` emits the values you already have. `Flux.range()` generates a sequence immediately. `Flux.interval()` generates a sequence over time, like a ticking clock.**

---

## Q: What are the use cases of `Flux.interval()`?

The simplest way to think about it is:

> Use `Flux.interval()` whenever you want to **do something repeatedly after a fixed amount of time**.

Instead of writing your own timer or scheduler, Reactor provides one for you.

---

# 1. Polling an external service (Most Common)

Suppose another service doesn't support events or Kafka notifications. The only way to know if data has changed is to **check periodically**.

Example:

Check order status every 5 seconds.

```java
Flux.interval(Duration.ofSeconds(5))
    .flatMap(i -> orderService.getOrderStatus())
    .subscribe(System.out::println);
```

Timeline

```
0s  ---- check order
5s  ---- check order
10s ---- check order
15s ---- check order
```

Real-world examples:

* Check payment status
* Check shipment status
* Check database for new records
* Check cache refresh

---

# 2. Refresh Dashboard

Imagine you're building a monitoring dashboard.

```
CPU Usage
Memory Usage
Requests/sec
```

You want fresh values every second.

```java
Flux.interval(Duration.ofSeconds(1))
    .flatMap(i -> metricsService.fetchMetrics())
```

Every second, new metrics are fetched.

---

# 3. Heartbeat Messages

Many systems send heartbeat messages to indicate they're still alive.

```
Server A
↓

"I'm Alive"

↓

every 30 seconds

↓

Server B
```

```java
Flux.interval(Duration.ofSeconds(30))
    .flatMap(i -> heartbeatService.send())
```

---

# 4. Retry Until Success

Suppose a database is temporarily unavailable.

Instead of failing permanently, retry every few seconds.

```java
Flux.interval(Duration.ofSeconds(5))
    .flatMap(i -> database.connect())
```

```
5 sec  -> Fail
10 sec -> Fail
15 sec -> Success
```

---

# 5. Periodic Cleanup

Delete expired sessions every hour.

```java
Flux.interval(Duration.ofHours(1))
    .flatMap(i -> sessionService.cleanup())
```

---

# 6. Cache Refresh

Instead of waiting until someone requests data, refresh it periodically.

```
Every 10 minutes

↓

Download latest exchange rates

↓

Update cache
```

---

# 7. Send Notifications

Example:

Every morning at 9 AM (combined with time calculations)

```
Check birthdays

↓

Send email

↓

Repeat tomorrow
```

---

# 8. Simulating Live Data

While developing, you may not have a real event source.

Generate one event every second.

```java
Flux.interval(Duration.ofSeconds(1))
    .map(i -> "Temperature : " + (20 + i))
```

Output

```
Temperature : 20
Temperature : 21
Temperature : 22
...
```

---

# 9. IoT Sensors

Imagine a temperature sensor.

Every second it sends a reading.

```
Sensor

↓

25°C

↓

26°C

↓

27°C
```

This naturally maps to

```java
Flux.interval(Duration.ofSeconds(1))
```

---

# 10. Stock Prices

Every second fetch the latest stock price.

```
1 sec

↓

₹120

↓

₹121

↓

₹119
```

---

# Spring WebFlux Example

Imagine you expose a streaming endpoint.

```java
@GetMapping(value = "/time", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
public Flux<String> time() {
    return Flux.interval(Duration.ofSeconds(1))
               .map(i -> LocalTime.now().toString());
}
```

The client continuously receives:

```
10:00:01

10:00:02

10:00:03

10:00:04
```

without making a new HTTP request each time.

---

# Why not use a `while(true)` loop?

A traditional approach might look like this:

```java
while (true) {
    doSomething();
    Thread.sleep(1000);
}
```

Problems:

* Blocks a thread while sleeping.
* Harder to compose with other reactive operations.
* Doesn't support backpressure.
* Cancellation and error handling require manual code.

With Reactor:

```java
Flux.interval(Duration.ofSeconds(1))
    .flatMap(i -> doSomething())
```

Benefits:

* Non-blocking.
* Easy to combine with other `Flux`/`Mono` operators.
* Easy to cancel (`take()`, `dispose()`).
* Fits naturally into a reactive pipeline.

---

# When should you use `Flux.interval()`?

Use it whenever you need a **continuous stream of timed events**, such as:

| Scenario                                | Use `Flux.interval()`?                           |
| --------------------------------------- | ------------------------------------------------ |
| Poll an API every few seconds           | ✅ Yes                                            |
| Refresh dashboard periodically          | ✅ Yes                                            |
| Send heartbeat messages                 | ✅ Yes                                            |
| Retry until a service becomes available | ✅ Yes                                            |
| Simulate live event streams             | ✅ Yes                                            |
| Schedule periodic cleanup               | ✅ Yes                                            |
| Generate one value immediately and stop | ❌ No, use `Mono.just()` or `Flux.just()`         |
| Emit a fixed list of values             | ❌ No, use `Flux.just()` or `Flux.fromIterable()` |
| Emit a range of numbers immediately     | ❌ No, use `Flux.range()`                         |

### Simple mental model

* **`Flux.just()`** → "I already have the data."
* **`Flux.range()`** → "Generate these numbers now."
* **`Flux.interval()`** → "Generate an event every *N* seconds."

Think of `Flux.interval()` as **a non-blocking timer that produces a stream of ticks**, where each tick can trigger any work you want to perform periodically.

---

## Q: In the `Flux.interval()` examples above, sometimes you used `map()` and sometimes `flatMap()`. What is the logic?

Excellent observation. This is one of the most important concepts in Reactor.

The decision has **nothing to do with `Flux.interval()`**. It depends entirely on **what your lambda returns**.

---

# Rule to remember

> **If your lambda returns a normal object → use `map()`**
>
> **If your lambda returns a `Mono` or `Flux` → use `flatMap()`**

That's it.

---

# Case 1: `map()`

Suppose `Flux.interval()` emits

```text
0
1
2
3
```

Now you simply convert each number into a String.

```java
Flux.interval(Duration.ofSeconds(1))
    .map(i -> "Tick " + i);
```

Here the lambda returns

```java
String
```

NOT

```java
Mono<String>
Flux<String>
```

So use `map()`.

Output

```text
Tick 0
Tick 1
Tick 2
```

---

# Another `map()` example

```java
Flux.interval(Duration.ofSeconds(1))
    .map(i -> i * 10);
```

Returns

```java
Long
```

Output

```text
0
10
20
30
```

Again, use `map()`.

---

# Case 2: `flatMap()`

Suppose you call a service.

```java
Mono<Order> getOrder(Long id)
```

Notice the return type.

It returns

```java
Mono<Order>
```

not

```java
Order
```

Now write

```java
Flux.interval(Duration.ofSeconds(1))
    .map(i -> orderService.getOrder(i));
```

What is the type now?

`Flux.interval()` emits

```text
0
1
2
```

Each becomes

```java
Mono<Order>
Mono<Order>
Mono<Order>
```

So the result is

```java
Flux<Mono<Order>>
```

which is almost never what you want.

---

Instead use

```java
Flux.interval(Duration.ofSeconds(1))
    .flatMap(i -> orderService.getOrder(i));
```

Now Reactor subscribes to each inner `Mono` and merges the results.

Result

```java
Flux<Order>
```

Exactly what you want.

---

# Visual Example

## Using `map()`

```java
Flux.interval(...)
```

emits

```text
0
1
2
```

Lambda

```java
i -> "Tick " + i
```

returns

```text
Tick 0
Tick 1
Tick 2
```

Pipeline

```text
Flux<Long>

↓

map()

↓

Flux<String>
```

---

## Using `flatMap()`

Lambda

```java
i -> orderService.getOrder(i)
```

returns

```text
Mono<Order>
Mono<Order>
Mono<Order>
```

Without `flatMap`

```text
Flux<Long>

↓

map()

↓

Flux<Mono<Order>>
```

With `flatMap`

```text
Flux<Long>

↓

flatMap()

↓

Flux<Order>
```

The extra layer is flattened away.

---

# Simple Analogy

Imagine a factory.

### `map()`

Worker receives

```
Apple
```

Worker returns

```
Juice
```

One input → One normal output.

```
Apple

↓

Juice
```

---

### `flatMap()`

Worker receives

```
Apple
```

Worker says

> "I'll ask another factory."

That other factory produces

```
Box containing Juice
```

```
Apple

↓

Box<Juice>
```

If you use `map()`, you get

```
Truck

↓

Box<Juice>
```

A truck carrying boxes.

If you use `flatMap()`, Reactor opens the boxes.

```
Truck

↓

Juice
```

---

# Why service calls usually use `flatMap()`

Reactive service methods almost always return `Mono` or `Flux`.

Example

```java
Mono<User> findUser(long id)
Mono<Order> getOrder(long id)
Flux<Product> getProducts()
```

Since they already return reactive types, use

```java
.flatMap(...)
```

---

# Why transformations usually use `map()`

Simple calculations don't return reactive types.

```java
.map(String::toUpperCase)

.map(i -> i * 10)

.map(User::getName)

.map(order -> order.getPrice())
```

Each returns a plain value.

---

# Real Examples

### Example 1 (map)

```java
Flux.interval(Duration.ofSeconds(1))
    .map(i -> "Second : " + i);
```

Returns

```java
String
```

Use

```java
map()
```

---

### Example 2 (flatMap)

```java
Flux.interval(Duration.ofSeconds(5))
    .flatMap(i -> paymentService.checkStatus());
```

Suppose

```java
checkStatus()
```

returns

```java
Mono<PaymentStatus>
```

Use

```java
flatMap()
```

---

### Example 3 (map)

```java
Flux.interval(Duration.ofSeconds(1))
    .map(i -> LocalTime.now());
```

`LocalTime.now()` returns

```java
LocalTime
```

Use

```java
map()
```

---

### Example 4 (flatMap)

```java
Flux.interval(Duration.ofSeconds(10))
    .flatMap(i -> webClient.get()
                           .retrieve()
                           .bodyToMono(String.class));
```

`bodyToMono()` returns

```java
Mono<String>
```

Use

```java
flatMap()
```

---

## The easiest rule to remember

Before writing either operator, ask:

> **What does my lambda return?**

* Returns `String`, `Integer`, `Order`, `User`, `LocalTime` (a plain object)? → **`map()`**
* Returns `Mono<T>` or `Flux<T>` (a reactive publisher)? → **`flatMap()`**

### Java analogy

Think of `map()` as calling a regular Java method:

```java
String getName(User user)
```

One object in, one object out.

Think of `flatMap()` as calling an asynchronous/reactive method:

```java
Mono<User> findUser(long id)
```

Since the method already gives you a `Mono`, `flatMap()` unwraps it so your pipeline continues with `User` values instead of `Mono<User>` values.

---

## Q: So does that mean generally for the usability of `map()` and `flatMap()` operators we should see the same logic as you have given? Can I apply the same rule below with any nuance?

> **Rule to remember**
>
> * If your lambda returns a normal object → use `map()`
> * If your lambda returns a `Mono` or `Flux` → use `flatMap()`

**Yes. This is the primary rule, and it works in about 95% of cases in Reactor.** It's the first thing every Reactor developer checks.

However, there are a few important nuances you should know.

---

# The Golden Rule

Always ask yourself:

> **"What does my lambda return?"**

### Returns a normal object?

```java
.map(user -> user.getName())
```

Returns:

```java
String
```

Use:

```java
map()
```

---

### Returns a reactive type?

```java
.flatMap(id -> userService.findUser(id))
```

Returns:

```java
Mono<User>
```

Use:

```java
flatMap()
```

---

# Why?

Because `map()` simply wraps whatever you return.

Suppose:

```java
Mono<User> findUser(Long id)
```

If you do:

```java
Flux<Long> ids = Flux.just(1L, 2L, 3L);

Flux<Mono<User>> users =
    ids.map(id -> userService.findUser(id));
```

The result is:

```java
Flux<
    Mono<User>,
    Mono<User>,
    Mono<User>
>
```

Notice the nesting.

You now have a stream of Monos, not a stream of Users.

---

`flatMap()` removes that extra layer.

```java
Flux<User> users =
    ids.flatMap(id -> userService.findUser(id));
```

Now you get

```java
User
User
User
```

instead of

```java
Mono<User>
Mono<User>
Mono<User>
```

---

# Nuance #1: `flatMap()` is not only about flattening

Many beginners think:

> "`flatMap()` is just `map()` + flatten."

That's true, but **it also subscribes to the inner publishers** and merges their emitted values into the outer stream.

For example:

```java
Flux.just(1, 2, 3)
    .flatMap(i -> Mono.just(i * 10));
```

Result:

```text
10
20
30
```

You don't have to manually subscribe to each `Mono`. `flatMap()` does that for you.

---

# Nuance #2: `flatMap()` may execute concurrently

This is a very important difference.

Suppose every service call takes 2 seconds.

```java
Flux.just(1, 2, 3)
    .flatMap(id -> userService.findUser(id));
```

Reactor can execute them **at the same time**.

Timeline

```text
Start User 1
Start User 2
Start User 3

↓

Results arrive whenever they finish.
```

Because of that, **the order of results is not guaranteed.**

Example output:

```text
User 2
User 1
User 3
```

---

# Nuance #3: Use `concatMap()` if order matters

If you want:

```text
1
2
3
```

always in order,

use

```java
.concatMap(...)
```

instead of

```java
.flatMap(...)
```

---

# Nuance #4: Use `flatMapMany()` only when starting from a `Mono`

Suppose

```java
Mono<List<User>>
```

or

```java
Mono<User>
```

needs to become a `Flux`.

You'll sometimes see

```java
.flatMapMany(...)
```

This is a specialized operator.

Don't worry about it yet.

---

# Nuance #5: Sometimes `map()` returns another `Mono` intentionally

Occasionally you'll see

```java
Flux<Mono<User>>
```

This is valid.

For example, you're building a pipeline that intentionally stores Monos for later processing.

In such rare cases,

```java
.map(...)
```

is correct.

But in normal application code, you almost never want nested publishers.

---

# Interview Rule

If an interviewer asks:

> **When should you use `map()` and when should you use `flatMap()`?**

A solid answer is:

> Use `map()` when the transformation returns a plain object. Use `flatMap()` when the transformation returns another reactive publisher (`Mono` or `Flux`), because `flatMap()` subscribes to the inner publisher and flattens its emitted values into the outer stream.

---

# Your Mental Decision Tree

```text
What does my lambda return?

                │
                ▼

      Plain Java Object?
      (String, User, Integer...)

             │
        YES  ▼
         map()

             │
        NO   ▼

      Mono<T> or Flux<T>?

             │
        YES  ▼
       flatMap()

             │
             ▼

Does the order of results matter?

        YES  → concatMap()

        NO   → flatMap()
```

## One refinement to your rule

Your rule is excellent for day-to-day coding. I would just expand it slightly:

> * **If the lambda returns a plain value (`T`) → use `map()`.**
> * **If the lambda returns a reactive publisher (`Mono<T>` or `Flux<T>`) and you want the emitted values → use `flatMap()`.**
> * **If order must be preserved while flattening → consider `concatMap()`.**

For someone learning Reactor, this refined rule will guide you correctly in the vast majority of real-world scenarios.
