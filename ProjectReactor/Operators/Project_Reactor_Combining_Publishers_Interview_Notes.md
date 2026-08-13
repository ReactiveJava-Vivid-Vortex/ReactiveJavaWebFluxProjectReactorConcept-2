# Combining Publishers in Project Reactor

## Index / Cheat Sheet

- [1. The Interview Problem](#1-the-interview-problem)
- [2. Quick Comparison](#2-quick-comparison)
- [3. `startWith`](#3-startwith)
- [4. `concatWith`](#4-concatwith)
- [5. `concatDelayError`](#5-concatdelayerror)
- [6. `merge`](#6-merge)
- [7. `zip`](#7-zip)
- [8. `flatMap` — Mono and Flux](#8-flatmap--mono-and-flux)
- [9. `concatMap`](#9-concatmap)
- [10. `collectList`](#10-collectlist)
- [11. `then`](#11-then)
- [12. Interview Decision Guide](#12-interview-decision-guide)
- [13. One-Page Mental Model](#13-one-page-mental-model)

---

# 1. The Interview Problem

A common interview question is:

> **"Let's say we have two separate reactive streams. We want to combine them. How would you do it in Project Reactor?"**

There is **no single answer**.

The correct operator depends on **what "combine" means**.

For example:

```java
Flux<String> stream1 = Flux.just("A", "B");
Flux<String> stream2 = Flux.just("C", "D");
```

You might want:

- `A B C D` → `concatWith`
- `A B` first, then `C D` → `concatWith`
- `A B C D`, but subscribe to both and interleave values → `merge`
- `A C`, `B D` → `zip`
- Add values before an existing stream → `startWith`
- Transform each value into another `Publisher` and combine the results → `flatMap`
- Transform each value into another `Publisher`, but preserve order → `concatMap`
- Wait for both/another publisher to finish and only signal completion → `then`
- Collect all values into one `List` → `collectList`

So, in an interview, don't simply say:

> "I would use `merge`."

Instead say:

> **"It depends on the required ordering and whether the streams should run sequentially, concurrently, or pair their values."**

That is the important interview answer.

---

# 2. Quick Comparison

| Operator | Main purpose | Ordering | Output |
|---|---|---|---|
| `startWith` | Put another Publisher/value before this stream | First Publisher/value always first | Same type |
| `concatWith` | Run Publishers sequentially | Preserves Publisher order | Same type |
| `concatDelayError` | Sequential concatenation, but delay errors | Preserves Publisher order | Same type |
| `merge` | Subscribe to Publishers concurrently and merge emissions | Not guaranteed | Same type |
| `zip` | Combine values position-by-position | Deterministic pairing | Combined type |
| `flatMap` | Transform each value into a Publisher and merge inner Publishers | Not guaranteed | Flattened type |
| `concatMap` | Transform each value into a Publisher sequentially | Preserves order | Flattened type |
| `collectList` | Aggregate all values into one List | Preserves emitted order | `Mono<List<T>>` |
| `then` | Ignore values and wait for completion | N/A | `Mono<Void>` or next Publisher |

### The easiest interview classification

```text
Need to combine Publishers?
        |
        +-- Sequential? ----------------> concatWith
        |
        +-- Sequential + delay errors? -> concatDelayError
        |
        +-- Concurrent/interleaved? ---> merge
        |
        +-- Pair values by position? --> zip
        |
        +-- Add something before? -----> startWith
        |
        +-- Transform each item to Publisher?
        |       |
        |       +-- Concurrent -------> flatMap
        |       |
        |       +-- Sequential -------> concatMap
        |
        +-- Need one List at the end? -> collectList
        |
        +-- Only care about completion? -> then
```

---

# 3. `startWith`

## Simple meaning

`startWith` means:

> **"Put this value/Publisher before my existing Publisher."**

It is useful when you want to prepend data.

### Example

```java
Flux<String> stream = Flux.just("B", "C");

Flux<String> result =
        stream.startWith("A");
```

Output:

```text
A
B
C
```

### With another Publisher

```java
Flux<String> first = Flux.just("A", "B");
Flux<String> second = Flux.just("C", "D");

Flux<String> result = second.startWith(first);
```

Output:

```text
A
B
C
D
```

The `first` Publisher is emitted before `second`.

## Interview example

**Interviewer:**

> "I already have a stream of database results, but I need to emit a default/header value before those results. What would you use?"

**Answer:**

> "I would use `startWith` because I need to prepend values or another Publisher before the existing stream."

## Important distinction

`startWith` is essentially **prepend**.

```text
startWith
   ↓
[A B] + [C D]
   ↓
A B C D
```

---

# 4. `concatWith`

## Simple meaning

`concatWith` means:

> **"Complete the first Publisher, then subscribe to the next Publisher."**

The Publishers run **sequentially**.

### Example

```java
Flux<String> first = Flux.just("A", "B");
Flux<String> second = Flux.just("C", "D");

Flux<String> result = first.concatWith(second);
```

Output:

```text
A
B
C
D
```

The important point is:

```text
first completes
      ↓
second starts
```

## Interview example

**Interviewer:**

> "You have two streams. The second stream must start only after the first stream finishes. Which operator?"

**Answer:**

> "`concatWith`, because it subscribes to the second Publisher only after the first Publisher completes and preserves their order."

## Real-world example

Imagine:

```text
Stream 1 = Load cached records
Stream 2 = Load database records
```

If you specifically want:

```text
cache records
      ↓
database records
```

then `concatWith` is a natural choice.

---

# 5. `concatDelayError`

## Simple meaning

`concatDelayError` means:

> **"Run Publishers sequentially, but don't immediately propagate an error from an earlier Publisher. Continue with the remaining Publishers and report the error later."**

Normal `concatWith`:

```text
Publisher 1
    ↓
ERROR
    ↓
STOP
```

`concatDelayError`:

```text
Publisher 1
    ↓
ERROR recorded
    ↓
Publisher 2
    ↓
Publisher 3
    ↓
ERROR propagated
```

### Example

```java
Flux<String> first =
        Flux.just("A")
            .concatWith(Flux.error(new RuntimeException("failed")));

Flux<String> second =
        Flux.just("B", "C");

Flux<String> result =
        first.concatWith(second);
```

With normal concatenation, the error prevents the later Publisher from being processed.

Conceptually, `concatDelayError` allows the later Publisher(s) to continue before the error is finally propagated.

## Interview example

**Interviewer:**

> "You need sequential processing, but an error from one Publisher should not immediately prevent the remaining Publishers from running. What would you use?"

**Answer:**

> "`concatDelayError` because it preserves sequential concatenation while delaying error propagation."

---

# 6. `merge`

## Simple meaning

`merge` means:

> **"Subscribe to multiple Publishers and emit their values as they arrive."**

Unlike `concatWith`, the Publishers do not have to wait for each other.

### Example

```java
Flux<String> first =
        Flux.just("A", "B")
            .delayElements(Duration.ofMillis(100));

Flux<String> second =
        Flux.just("1", "2");

Flux<String> result =
        Flux.merge(first, second);
```

Possible output:

```text
1
2
A
B
```

or:

```text
A
1
2
B
```

The exact order depends on when values are emitted.

## Key point

`merge` is about **concurrent/interleaved emission**.

```text
Stream 1 ── A ───── B ───────
             \       \
Stream 2 ───── 1 ─ 2 ─────────
              ↓
           merge
              ↓
        A 1 B 2 ...
```

## Interview example

**Interviewer:**

> "I have two independent API calls. I want to subscribe to both and process whichever response arrives first. What would you use?"

**Answer:**

> "`merge`, because the two Publishers can run independently and their emissions can be interleaved as they arrive."

### `concatWith` vs `merge`

This is one of the most important interview comparisons.

```text
concatWith:

A B | C D
----|----
first   second


merge:

A       B
 \     /
  \   /
   C D
```

| Requirement | Operator |
|---|---|
| First completely, then second | `concatWith` |
| Both can emit independently | `merge` |
| Preserve source order | `concatWith` |
| Order not guaranteed | `merge` |

---

# 7. `zip`

## Simple meaning

`zip` means:

> **"Take one value from each Publisher and combine them into one value."**

Think of it as **pairing values by position**.

### Example

```java
Flux<String> names =
        Flux.just("John", "David", "Alex");

Flux<Integer> ages =
        Flux.just(20, 30, 40);

Flux<String> result =
        Flux.zip(names, ages)
            .map(tuple -> tuple.getT1() + ":" + tuple.getT2());
```

Output:

```text
John:20
David:30
Alex:40
```

Conceptually:

```text
names:  John   David   Alex
          |      |      |
ages:    20     30     40
          |      |      |
          +------+------+ 
                 ↓
             zip
                 ↓
        (John,20)
        (David,30)
        (Alex,40)
```

## Interview example

**Interviewer:**

> "I have two streams: customer names and customer ages. I need to combine the first name with the first age, second name with second age, and so on. What would you use?"

**Answer:**

> "`zip`, because I need position-based pairing rather than simply concatenating the streams."

## Important difference: `merge` vs `zip`

This is a very common interview question.

### `merge`

```text
A B C
1 2 3

Possible:

A
1
B
2
3
C
```

It combines **emissions**.

### `zip`

```text
A  B  C
|  |  |
1  2  3

↓

(A,1)
(B,2)
(C,3)
```

It combines **corresponding values**.

---

# 8. `flatMap` — Mono and Flux

## Simple meaning

`flatMap` means:

> **"For every item, create another Publisher, then flatten all those Publishers into one Publisher."**

This is extremely important in reactive programming.

---

## 8.1 `Flux.flatMap`

Suppose:

```java
Flux<String> users =
        Flux.just("John", "David", "Alex");
```

For every user, call an API:

```java
Flux<UserDetails> result =
        users.flatMap(user -> getUserDetails(user));
```

Where:

```java
Mono<UserDetails> getUserDetails(String user)
```

The important part is:

```text
Flux<User>
    |
    +-- John  -> Mono<UserDetails>
    |
    +-- David -> Mono<UserDetails>
    |
    +-- Alex  -> Mono<UserDetails>
              |
              ↓
            flatMap
              ↓
        Flux<UserDetails>
```

`flatMap` is therefore often used when a value needs to be transformed into another asynchronous Publisher.

### Interview example

**Interviewer:**

> "You have a Flux of user IDs. For every ID, you need to call another asynchronous service that returns Mono<User>. How would you do it?"

**Answer:**

> "I would use `flatMap`, because each ID needs to be transformed into a `Mono<User>`, and `flatMap` flattens those Monos into one Flux."

---

## 8.2 `Mono.flatMap`

Suppose one user ID produces one user:

```java
Mono<String> userId = Mono.just("101");

Mono<User> result =
        userId.flatMap(id -> getUser(id));
```

Where:

```java
Mono<User> getUser(String id)
```

Conceptually:

```text
Mono<String>
    |
    | flatMap
    ↓
Mono<User>
```

This is very common in WebFlux service-layer code.

### Example

```java
Mono<User> getUser(String id) {
    return repository.findById(id);
}

Mono<UserDetails> result =
        userService.getUser("101")
                   .flatMap(user -> getUserDetails(user));
```

Here the first operation returns:

```text
Mono<User>
```

and the next asynchronous operation returns:

```text
Mono<UserDetails>
```

`flatMap` prevents you from ending up with:

```text
Mono<Mono<UserDetails>>
```

Instead, you get:

```text
Mono<UserDetails>
```

---

## `flatMap` ordering

`flatMap` does **not guarantee the original order** when asynchronous inner Publishers complete at different times.

Example:

```text
A → slow API
B → fast API
C → medium API
```

Possible result:

```text
B
C
A
```

If order matters, consider `concatMap`.

---

# 9. `concatMap`

## Simple meaning

`concatMap` is similar to `flatMap`, but it processes the inner Publishers **sequentially and preserves order**.

### Example

```java
Flux<String> users =
        Flux.just("A", "B", "C");

Flux<UserDetails> result =
        users.concatMap(user -> getUserDetails(user));
```

Conceptually:

```text
A
 ↓
API call
 ↓
complete
 ↓
B
 ↓
API call
 ↓
complete
 ↓
C
```

The output order remains:

```text
A
B
C
```

assuming each mapping produces the corresponding result.

## Interview example

**Interviewer:**

> "You have a Flux of IDs. For every ID you need to call an asynchronous API, but the results must remain in the same order as the IDs. Would you use flatMap?"

**Answer:**

> "I would prefer `concatMap` when sequential processing and ordering are required. `flatMap` allows inner Publishers to run concurrently and does not guarantee ordering."

## `flatMap` vs `concatMap`

| Requirement | `flatMap` | `concatMap` |
|---|---|---|
| Transform each item to Publisher | Yes | Yes |
| Concurrent inner Publishers | Yes | No |
| Sequential inner Publishers | No | Yes |
| Preserve order | Not guaranteed | Yes |
| Good for independent async calls | Yes | Less suitable |
| Good when order matters | No | Yes |

---

# 10. `collectList`

## Simple meaning

`collectList` means:

> **"Collect all emitted values into one List and emit that List as a Mono."**

### Example

```java
Flux<String> names =
        Flux.just("John", "David", "Alex");

Mono<List<String>> result =
        names.collectList();
```

Result:

```text
["John", "David", "Alex"]
```

The important type change is:

```text
Flux<String>
      ↓
collectList()
      ↓
Mono<List<String>>
```

## Interview example

**Interviewer:**

> "You receive multiple records from a Flux, but your next operation needs all records together as a List. What would you use?"

**Answer:**

> "`collectList`, because it aggregates all values from the Flux and emits them as a single `Mono<List<T>>`."

## Important warning

`collectList` waits for the entire Flux to complete.

So this:

```java
Flux.never()
    .collectList();
```

will never produce a List.

Also, collecting a very large/unbounded stream can consume a lot of memory.

---

# 11. `then`

## Simple meaning

`then` means:

> **"I don't care about the values. I only care that the Publisher completes, and then I want to continue."**

Example:

```java
Mono<Void> result =
        saveUser()
            .then();
```

If `saveUser()` emits:

```text
User
```

`then()` ignores the `User` value.

It only waits for completion.

```text
saveUser()
    |
    | User
    | User
    | User
    ↓
 complete
    ↓
 then()
    ↓
 Mono<Void>
```

---

## `then` with another Publisher

This is particularly useful for sequencing operations.

```java
Mono<String> result =
        saveUser()
            .then(loadUser());
```

Meaning:

```text
saveUser()
    ↓
wait for completion
    ↓
loadUser()
    ↓
emit its result
```

The values from `saveUser()` are ignored.

## Interview example

**Interviewer:**

> "You need to save something first. You don't care about the save result. Once saving completes, you need to execute another reactive operation and return its result. What would you use?"

**Answer:**

> "I would use `then` because I need to wait for completion of the first Publisher, ignore its emitted value, and then continue with another Publisher."

---

# 12. Interview Decision Guide

When the interviewer says **"combine two Publishers"**, ask yourself what they actually mean.

### Question 1: Should Publisher 2 wait for Publisher 1?

**Yes**

```java
first.concatWith(second)
```

Use:

> `concatWith`

---

### Question 2: Should errors be delayed while concatenating?

**Yes**

```java
first.concatWith(second) // immediate error
```

vs the appropriate `concatDelayError` form.

Use:

> `concatDelayError`

---

### Question 3: Should both Publishers run independently and emit as values arrive?

**Yes**

```java
Flux.merge(first, second)
```

Use:

> `merge`

---

### Question 4: Do I need to pair corresponding values?

**Yes**

```java
Flux.zip(first, second)
```

Use:

> `zip`

Think:

```text
first value  + second value
first value  + second value
first value  + second value
```

---

### Question 5: Do I need to add something before an existing Publisher?

**Yes**

```java
stream.startWith(value)
```

Use:

> `startWith`

---

### Question 6: Does each item produce another Publisher?

**Yes**

```java
flux.flatMap(item -> asyncOperation(item))
```

Use:

> `flatMap`

when concurrent processing is desirable.

---

### Question 7: Does each item produce another Publisher, but order matters?

**Yes**

```java
flux.concatMap(item -> asyncOperation(item))
```

Use:

> `concatMap`

---

### Question 8: Do I need all values as one List?

**Yes**

```java
flux.collectList()
```

Use:

> `collectList`

---

### Question 9: Do I only care that an operation completes?

**Yes**

```java
publisher.then()
```

or:

```java
publisher.then(nextPublisher)
```

Use:

> `then`

---

# 13. One-Page Mental Model

The easiest way to remember these operators is to think about **what you want to do with time, order, and values**.

```text
                 COMBINING / CHAINING
                         |
        +----------------+----------------+
        |                |                |
      ORDER            TIME            VALUES
        |                |                |
        |                |                |
   concatWith         merge             zip
        |                |                |
 "one after           "as they        "pair them"
  another"             arrive"
        |
 concatDelayError
 "one after another,
  delay errors"


startWith
"put this BEFORE"


flatMap
"map each item to a Publisher
 and run inner Publishers
 independently"


concatMap
"map each item to a Publisher
 and process sequentially"


collectList
"give me ALL values together"


then
"I don't care about values;
 tell me when it is COMPLETE"
```

## Final interview cheat sheet

```text
PREPEND
startWith

SEQUENTIAL
concatWith

SEQUENTIAL + DELAY ERROR
concatDelayError

CONCURRENT / INTERLEAVED
merge

PAIR VALUES
zip

ASYNC TRANSFORMATION + CONCURRENT
flatMap

ASYNC TRANSFORMATION + ORDER
concatMap

COLLECT EVERYTHING
collectList

IGNORE VALUES, WAIT FOR COMPLETION
then
```

## Most important interview distinction

If asked:

> **"How do you combine two streams?"**

Don't immediately answer with one operator.

Say:

> **"It depends on the required semantics. If I need sequential execution, I'd use `concatWith`; if I want independent/concurrent emissions, `merge`; if I need position-based pairing, `zip`; and if each item needs to trigger another asynchronous Publisher, I'd use `flatMap` or `concatMap` depending on whether ordering is required."**

That answer demonstrates that you understand the **semantics**, rather than simply memorizing operators.
