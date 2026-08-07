# Q1: What is `repeat` in Reactor?

## Simple Meaning

**Repeat = run the same sequence again after it completes successfully**

👉 Think:

> “Do this again after finishing”

---

## Example (Simple)

```java
Flux.just("A", "B")
    .repeat(2)
    .subscribe(System.out::println);
```

### Output:

```
A B
A B
A B
```

## Explanation

* Original run → `A B`
* Repeat 2 times → runs 2 more times
* Total runs = **1 + 2 = 3**

---

## Key Points

* Works only when stream **completes normally**
* Does NOT care about errors
* Used when you want to **loop successful flow**

---

## Real-Life Example

Polling API every few seconds:

```java
Flux.fromCallable(() -> fetchData())
    .repeat()
```

---

# Q2: What is `retry` in Reactor?

## Simple Meaning

**Retry = run the sequence again only if it fails (error happens)**

👉 Think:

> “Try again if something goes wrong”

---

## Example (Simple)

```java
Flux.just(1, 2, 0)
    .map(i -> 10 / i)
    .retry(2)
    .subscribe(System.out::println);
```

### Output:

```
10
5
ERROR after retries
```

## Explanation

* First run:

  * 10/1 → OK
  * 10/2 → OK
  * 10/0 → ❌ Error
* Retry again (2 times)
* If still fails → final error

---

## Key Points

* Works only on **errors**
* Re-subscribes to the source
* Useful for **unstable operations** (API, DB, network)

---

## Real-Life Example

Retry API call if it fails:

```java
Mono.fromCallable(() -> callApi())
    .retry(3)
```

---

# Q3: Difference between `repeat` vs `retry`

| Feature        | repeat                 | retry                   |
| -------------- | ---------------------- | ----------------------- |
| Trigger        | On completion          | On error                |
| Purpose        | Loop success flow      | Recover from failure    |
| Use case       | Polling, periodic jobs | API retries, resilience |
| Error handling | Ignores errors         | Handles errors          |

---

# Q4: Important Nuance (VERY IMPORTANT)

### ⚠️ Both `repeat` and `retry` **re-subscribe**

This means:

* Entire flow starts again from beginning
* Side effects can happen again (DB call, API call)

---

### ⚠️ Infinite Loop Risk

```java
.repeat()
.retry()
```

👉 Can run forever if not controlled

---

### Better Control (Advanced)

```java
.retryWhen(Retry.fixedDelay(3, Duration.ofSeconds(2)))
```

---

# Final Mental Model

* **repeat → success loop**
* **retry → failure recovery**

---

# Quick Analogy

* `repeat` = Watching a movie again because you liked it 🎬
* `retry` = Restarting the movie because it crashed ⚠️

---

If you want next level understanding, I can explain:

* `retryWhen` vs `retry`
* `repeatWhen`
* Real production patterns (very important for interviews + system design)

Just tell me 👍
