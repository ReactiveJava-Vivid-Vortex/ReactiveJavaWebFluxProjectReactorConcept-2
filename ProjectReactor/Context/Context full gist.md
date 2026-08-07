# ❓ Q1: What is Context in Reactor?

## ✅ Simple Explanation

Think of **Context** as a **hidden map (key-value store)** that travels along with your reactive pipeline.

👉 It is like:

```java
Map<String, Object> context;
```

But instead of passing it manually everywhere, Reactor **carries it internally**.

---

## 🎯 Real-life analogy

Imagine a **request in a microservice**:

* userId = "123"
* requestId = "abc-xyz"
* authToken = "jwt-token"

Instead of passing these in every method:

```java
method1(userId, requestId)
method2(userId, requestId)
```

👉 You store it in **Context**, and Reactor automatically carries it.

---

## ⚠️ Important Nuances (VERY IMPORTANT)

### 1. Context is **immutable**

Every update creates a **new Context**, not modify existing.

### 2. It flows **from subscriber → upstream (reverse direction)**

👉 This is the most confusing part:

* Data flows **downstream**
* Context flows **upstream**

---

### 3. Not same as ThreadLocal ❌

| Feature       | ThreadLocal | Reactor Context |
| ------------- | ----------- | --------------- |
| Thread based  | ✅           | ❌               |
| Reactive safe | ❌           | ✅               |
| Async safe    | ❌           | ✅               |

---

---

# ❓ Q2: Context Append / Update / Delete

---

## ✅ Append (Add new key)

```java
Mono<String> mono = Mono.deferContextual(ctx ->
    Mono.just("User: " + ctx.get("userId"))
);

mono.contextWrite(ctx -> ctx.put("userId", "123"))
    .subscribe(System.out::println);
```

### ✔ Output:

```
User: 123
```

👉 Here:

* `put()` adds new key

---

## ✅ Update (Override existing key)

```java
Mono<String> mono = Mono.deferContextual(ctx ->
    Mono.just("User: " + ctx.get("userId"))
);

mono
.contextWrite(ctx -> ctx.put("userId", "123"))
.contextWrite(ctx -> ctx.put("userId", "456")) // override
.subscribe(System.out::println);
```

### ✔ Output:

```
User: 456
```

👉 Latest value wins.

---

## ✅ Delete (Remove key)

```java
mono
.contextWrite(ctx -> ctx.put("userId", "123"))
.contextWrite(ctx -> ctx.delete("userId"))
.subscribe(System.out::println);
```

### ✔ Result:

👉 Will throw error (key not found)

---

## 🔥 Important Observation

Order matters because:
👉 Context flows **bottom → top**

---

---

# ❓ Q3: Context Propagation

---

## ✅ Simple Idea

Context automatically moves through operators **without passing manually**

---

## 🧠 Example

```java
Mono<String> mono =
    Mono.deferContextual(ctx ->
        Mono.just("User: " + ctx.get("userId"))
    );

mono
    .map(data -> data + " processed")
    .contextWrite(ctx -> ctx.put("userId", "123"))
    .subscribe(System.out::println);
```

### ✔ Output:

```
User: 123 processed
```

---

## ⚠️ Problem Scenario (Thread Switch)

```java
mono
    .publishOn(Schedulers.parallel())
    .contextWrite(ctx -> ctx.put("userId", "123"))
```

👉 Context still works ✅
👉 Unlike ThreadLocal ❌ (which would break)

---

## 🧠 Why this is powerful?

Because:

* Works across threads
* Works in async flows
* Perfect for logging, tracing

---

## 🔥 Real-world use cases

* Logging correlationId
* Security (user info)
* Multi-tenant apps
* Tracing (Zipkin, OpenTelemetry)

---

---

# ❓ Q4: Rate Limiter with Context (Full Explanation + Implementation)

---

## ✅ Problem Statement

You want to:

* Limit requests per user
* But user info is in **Context**

---

## 🧠 Idea

1. Store `userId` in Context
2. Fetch it during execution
3. Apply rate limit per user

---

---

## 🧩 Step 1: Simple Rate Limiter (in-memory)

```java
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.atomic.AtomicInteger;

class RateLimiter {
    private static final int LIMIT = 2;
    private final ConcurrentHashMap<String, AtomicInteger> map = new ConcurrentHashMap<>();

    public boolean allow(String userId) {
        map.putIfAbsent(userId, new AtomicInteger(0));
        return map.get(userId).incrementAndGet() <= LIMIT;
    }
}
```

---

## 🧩 Step 2: Use with Reactor Context

```java
RateLimiter limiter = new RateLimiter();

Mono<String> service =
    Mono.deferContextual(ctx -> {
        String userId = ctx.get("userId");

        if (!limiter.allow(userId)) {
            return Mono.error(new RuntimeException("Rate limit exceeded"));
        }

        return Mono.just("Processed for user " + userId);
    });
```

---

## 🧩 Step 3: Execute

```java
service
.contextWrite(ctx -> ctx.put("userId", "user1"))
.subscribe(System.out::println, System.err::println);

service
.contextWrite(ctx -> ctx.put("userId", "user1"))
.subscribe(System.out::println, System.err::println);

service
.contextWrite(ctx -> ctx.put("userId", "user1"))
.subscribe(System.out::println, System.err::println);
```

---

## ✔ Output

```
Processed for user user1
Processed for user user1
Rate limit exceeded
```

---

---

## 🔥 Important Improvements (Production Level)

### ❌ Problem with above:

* Memory leak (map grows)
* No time window

---

### ✅ Better Approach:

Use:

* Bucket4j
* Redis
* Token bucket algorithm

---

---

## 🧠 Advanced Version (Cleaner)

```java
Mono<String> service =
    Mono.deferContextual(ctx -> {
        String userId = ctx.getOrDefault("userId", "guest");

        return Mono.just(userId)
            .filter(limiter::allow)
            .switchIfEmpty(Mono.error(new RuntimeException("Rate limit exceeded")))
            .map(id -> "Processed for user " + id);
    });
```

---

---

# 🔥 Final Summary (Very Important)

### Context is:

* Immutable key-value store
* Flows opposite to data
* Replaces ThreadLocal in reactive world

---

### Key APIs:

| Operation | Method              |
| --------- | ------------------- |
| Read      | `deferContextual()` |
| Write     | `contextWrite()`    |
| Add       | `put()`             |
| Delete    | `delete()`          |

---

---

# 💡 Brutally Honest Advice (Important for your growth)

Since you're aiming for **Architect role**:

👉 You MUST understand:

* Context propagation with **Schedulers**
* Context in **WebFlux filters**
* Context vs MDC logging
* Context with **OpenTelemetry**

👉 This is asked in **senior interviews**

---

---

If you want next level:
👉 I can show:

* Context in **Spring WebFlux (real production)**
* Context + MDC logging integration
* Context debugging tricks

Just tell me 👍
