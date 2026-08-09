 Q: Is Reactor synchronous by default and becomes asynchronous only with schedulers?

**Answer:**
👉 **Yes… but with an important nuance.**

## Simple Explanation

* ✅ **By default, Reactor runs synchronously on the <mark>calling thread</mark>**
* ❗ But Reactor itself is **non-blocking by design**, even when synchronous

👉 <mark>When you add schedulers:<mark> This is when it schedules the task in another thread using schedular (based on schedular types).

```java
.subscribeOn(Schedulers.boundedElastic())
.publishOn(Schedulers.parallel())
```

* Now execution can **switch threads**
* This is what makes it **truly asynchronous**

---

## Key Clarification (Important)

* **Synchronous ≠ Blocking always**
* <mark>Reactor can be:</mark>

  * <mark>**Synchronous + Non-blocking** (**default case***)</mark>
  * **Asynchronous + Non-blocking** (with schedulers)

---

## Best Side Note (put this in your MD)

> **Note:** By default, Reactor executes synchronously on the calling thread. It becomes asynchronous only when schedulers are used. However, even in synchronous mode, Reactor remains non-blocking.

---

## Super Simple Mental Model

* No scheduler → same thread → looks synchronous
* With scheduler → different thread → async behavior

---

If you want next-level clarity, I can show you a **1 small code example where same Flux behaves sync vs async just by adding one line** — very useful for interviews.
