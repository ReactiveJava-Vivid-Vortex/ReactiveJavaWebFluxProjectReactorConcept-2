## ❓ Q1: What is **Sink emit failure handler** in simple terms?

### 👉 Simple Explanation

When you try to push data into a **Sink**, it **may fail** sometimes.

Instead of failing silently, Reactor asks:

👉 *“What should I do if emit fails?”*

That decision is handled by the **emit failure handler**.

---

## 🧠 Think of it like this

Imagine:

* Sink = **bucket**
* You = **pouring water**
* Sometimes bucket says: ❌ “I can’t take more!”

Now you have choices:

* Try again? 🔁
* Ignore? 😐
* Throw error? 💥

👉 That choice = **Emit Failure Handler**

---

## ❓ Q2: When does emit fail?

Emit can fail when:

* No subscribers are present
* Buffer is full (backpressure)
* Sink is already completed
* Concurrent threads emitting (race condition)

---

## ❓ Q3: Where do we use it?

When you use:

```java
sink.emitNext(value, handler);
```

👉 That `handler` decides what to do if emission fails.

---

## ❓ Q4: Common handlers (very important)

### 1. `FAIL_FAST`

👉 "If it fails, just throw error immediately"

```java
sink.emitNext(1, Sinks.EmitFailureHandler.FAIL_FAST);
```

✔ Simple
❌ No retry

---

### 2. `busyLooping()`

👉 "Keep retrying until it works"

```java
sink.emitNext(1, Sinks.EmitFailureHandler.busyLooping());
```

✔ Useful for race conditions
❌ Can consume CPU

---

### 3. Custom handler

```java
sink.emitNext(1, (signalType, emitResult) -> {
    return emitResult == Sinks.EmitResult.FAIL_NON_SERIALIZED;
});
```

👉 Retry only for specific failure

---

## ❓ Q5: Difference vs `tryEmitNext` (important)

| Method          | Behavior                                         |
| --------------- | ------------------------------------------------ |
| `tryEmitNext()` | Returns success/failure (you handle it manually) |
| `emitNext()`    | Uses **failure handler** to decide automatically |

---

## ⚡ Quick Summary

* `emitNext` may fail
* Failure handler tells Reactor **what to do next**
* Options:

  * Fail immediately
  * Retry
  * Custom logic

---

## 🧠 Pro Insight (Architect level thinking)

* Use `FAIL_FAST` in most cases → predictable behavior
* Use retry (`busyLooping`) only for **multi-thread race issues**
* Prefer `tryEmitNext` when you want **full control**

---

If you want next level clarity, I can show:

👉 Real-world example (event bus using Sink)
👉 When exactly each failure type happens internally

Just ask 👍
