# ✅ Q1. What is Threading & Schedulers (Simple Explanation)

### 🔹 Threading (very simple)

Thread = a worker doing your task.

👉 Example:

* Imagine a restaurant 🍽️

  * 1 chef = 1 thread
  * Multiple chefs = multiple threads

If only 1 chef → slow
If multiple chefs → faster

---

### 🔹 In normal Java

```java
new Thread(() -> System.out.println("Hello")).start();
```

👉 You manually create/manage threads.

---

### 🔹 In Reactor

You **DON’T manage threads manually**

👉 Reactor uses:
👉 **Schedulers = thread managers**

---

### 🔹 One-line understanding:

* **Threading = who is doing the work**
* **Scheduler = where (which thread pool) work runs**

---

# ✅ Q2. Default Thread Model (VERY IMPORTANT)

### 🔹 By default:

👉 Reactor runs on **same thread**

```java
Flux.just(1, 2, 3)
    .map(i -> {
        System.out.println(Thread.currentThread().getName());
        return i * 2;
    })
    .subscribe();
```

👉 Output:

```
main
main
main
```

---

### 🔥 Key Insight:

👉 **Reactor is NOT multi-threaded by default**

This is the biggest misconception.

---

# ✅ Q3. Thread Switching

### 🔹 What is it?

Changing execution from one thread to another.

---

### 🔹 Example:

```java
Flux.just(1, 2, 3)
    .publishOn(Schedulers.parallel())
    .map(i -> {
        System.out.println(Thread.currentThread().getName());
        return i;
    })
    .subscribe();
```

👉 Output:

```
parallel-1
parallel-1
parallel-1
```

---

### 🔥 Insight:

👉 Thread changes ONLY when you explicitly say so

---

# ✅ Q4. Scheduler Concept

### 🔹 Scheduler = Thread pool provider

👉 Think:

```
ExecutorService in Java = Scheduler in Reactor
```

---

### 🔹 Types:

* boundedElastic → blocking tasks
* parallel → CPU tasks
* single → one thread
* immediate → current thread

---

# ✅ Q5. boundedElastic()

### 🔹 Use for:

👉 Blocking / slow operations

---

### 🔹 Example:

```java
Mono.fromCallable(() -> {
    Thread.sleep(1000); // blocking
    return "data";
})
.subscribeOn(Schedulers.boundedElastic())
.subscribe(System.out::println);
```

---

### 🔥 Key Points:

* Creates threads dynamically
* Max limit exists (safe)
* Good for DB calls, file IO

---

# ✅ Q6. parallel()

### 🔹 Use for:

👉 CPU intensive work

---

### 🔹 Example:

```java
Flux.range(1, 5)
    .parallel()
    .runOn(Schedulers.parallel())
    .map(i -> {
        System.out.println(Thread.currentThread().getName());
        return i * i;
    })
    .sequential()
    .subscribe(System.out::println);
```

---

### 🔥 Key Points:

* Fixed thread pool
* Size = CPU cores
* Best for calculations

---

# ✅ Q7. single()

### 🔹 Use for:

👉 One dedicated thread

---

### 🔹 Example:

```java
Flux.range(1, 3)
    .publishOn(Schedulers.single())
    .map(i -> {
        System.out.println(Thread.currentThread().getName());
        return i;
    })
    .subscribe();
```

---

### 🔥 Use case:

* Logging
* Ordering guarantee
* Shared mutable state

---

# ✅ Q8. immediate()

### 🔹 Use for:

👉 No scheduling (same thread)

---

### 🔹 Example:

```java
Flux.just(1)
    .publishOn(Schedulers.immediate())
    .map(i -> {
        System.out.println(Thread.currentThread().getName());
        return i;
    })
    .subscribe();
```

---

### 🔥 Insight:

👉 Same as default behavior

---

# ✅ Q9. publishOn()

### 🔹 What it does:

👉 Changes thread for **downstream**

---

### 🔹 Example:

```java
Flux.just(1, 2, 3)
    .map(i -> {
        System.out.println("Before: " + Thread.currentThread().getName());
        return i;
    })
    .publishOn(Schedulers.parallel())
    .map(i -> {
        System.out.println("After: " + Thread.currentThread().getName());
        return i;
    })
    .subscribe();
```

---

### 🔥 Output:

```
Before: main
After: parallel-1
```

---

### 🔥 Rule:

👉 publishOn affects **below operators**

---

# ✅ Q10. subscribeOn()

### 🔹 What it does:

👉 Changes thread for **entire pipeline (source)**

---

### 🔹 Example:

```java
Flux.just(1)
    .map(i -> {
        System.out.println(Thread.currentThread().getName());
        return i;
    })
    .subscribeOn(Schedulers.parallel())
    .subscribe();
```

---

### 🔥 Rule:

👉 subscribeOn affects **upstream (whole flow)**

---

# 🔥 IMPORTANT DIFFERENCE

| Feature  | publishOn       | subscribeOn      |
| -------- | --------------- | ---------------- |
| Affects  | downstream      | upstream (whole) |
| Position | matters         | doesn't matter   |
| Use case | switch mid-flow | control source   |

---

# ✅ Q11. Thread Pools

### 🔹 Reactor provides:

* boundedElastic → dynamic pool
* parallel → fixed pool
* single → single thread

---

### 🔥 Important:

👉 You don’t create pools manually
👉 Use built-in schedulers

---

# ✅ Q12. CPU-bound work

### 🔹 Definition:

👉 Heavy computation

Examples:

* Sorting
* Encryption
* Calculations

---

### 🔹 Use:

```java
Schedulers.parallel()
```

---

# ✅ Q13. I/O-bound work

### 🔹 Definition:

👉 Waiting operations

Examples:

* DB calls
* API calls
* File read/write

---

### 🔹 Use:

```java
Schedulers.boundedElastic()
```

---

# ✅ Q14. Thread Affinity

### 🔹 Meaning:

👉 Same task stays on same thread

---

### 🔹 Example:

```java
Flux.range(1, 3)
    .publishOn(Schedulers.single())
    .map(i -> Thread.currentThread().getName())
    .subscribe(System.out::println);
```

---

👉 Output:

```
single-1
single-1
single-1
```

---

### 🔥 Why important?

* Avoid race conditions
* Maintain order

---

# ✅ Q15. Scheduler Best Practices (VERY IMPORTANT)

### 🔥 1. Don’t block parallel threads ❌

```java
Thread.sleep() inside parallel() → BAD
```

---

### 🔥 2. Use correct scheduler

* CPU → parallel
* Blocking → boundedElastic

---

### 🔥 3. Minimize thread switching

👉 Too many switches = overhead

---

### 🔥 4. Prefer non-blocking APIs

👉 Reactor is designed for async

---

### 🔥 5. Don’t overuse subscribeOn()

👉 Use it only once (usually)

---

### 🔥 6. Understand flow clearly

👉 Thread = where execution happens

---

# 🔥 FINAL MENTAL MODEL (VERY IMPORTANT)

Think like this:

```
Flux → Pipeline
Scheduler → Workers
publishOn → Switch worker mid-way
subscribeOn → Choose worker at start
```

---

# 💡 Brutally Honest Insight

If you don’t understand threading:
👉 You will misuse WebFlux
👉 You may accidentally block threads
👉 System will look reactive but behave like synchronous

---

# 🚀 Productivity Tip for You

When learning:
👉 Always print thread name:

```java
System.out.println(Thread.currentThread().getName());
```

This alone will make everything crystal clear.

---

If you want next:
👉 I can give you **real-world scenarios (DB + API + Kafka)**
👉 Or tricky interview questions on this topic
