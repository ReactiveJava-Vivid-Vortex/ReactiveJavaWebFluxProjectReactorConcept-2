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
