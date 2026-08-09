**clean, interview-ready cheat sheet** for **Hot vs Cold Publisher** 👇

---

# 📌 **Project Reactor – Hot vs Cold Publisher Cheat Sheet**

---

## 🔹 1. What is Cold Publisher?

👉 A **Cold Publisher** starts emitting **only when a subscriber subscribes**

👉 Each subscriber gets **its own data stream**

### 🧠 Simple Thinking:

> “Netflix – every user starts movie from beginning”

```java
Flux<String> flux = Flux.just("A", "B", "C");

flux.subscribe(System.out::println); // gets A B C
flux.subscribe(System.out::println); // again gets A B C
```

✔ Data is **replayed for every subscriber**
✔ Execution happens **per subscription**

---

## 🔹 2. What is Hot Publisher?

👉 A **Hot Publisher** emits data **independent of subscribers**

👉 Subscribers only get **data emitted after they subscribe**

### 🧠 Simple Thinking:

> “Live TV – you only see current broadcast”

```java
Sinks.Many<String> sink = Sinks.many().multicast().onBackpressureBuffer();

Flux<String> flux = sink.asFlux();

sink.tryEmitNext("A"); // no subscriber yet → lost

flux.subscribe(System.out::println);

sink.tryEmitNext("B"); // subscriber gets this
```

✔ No replay by default
✔ Shared stream among subscribers

---

## 🔹 3. Key Differences (Interview Table)

| Feature                  | Cold Publisher | Hot Publisher           |
| ------------------------ | -------------- | ----------------------- |
| Start                    | On subscribe   | Independent             |
| Data for each subscriber | Separate       | Shared                  |
| Replay                   | Yes            | No (by default)         |
| Example                  | `Flux.just()`  | `Sinks.Many`, `share()` |
| Use case                 | DB/API calls   | Live events             |

---

## 🔹 4. Converting Cold → Hot

### 🔸 Using `share()`

```java
Flux<Integer> flux = Flux.range(1, 3).share();
```

✔ Now behaves like hot
✔ Multiple subscribers share same execution

---

### 🔸 Using `publish().refCount()`

```java
Flux<Integer> flux = Flux.range(1, 3)
        .publish()
        .refCount(1);
```

✔ Starts when first subscriber comes
✔ Stops when all unsubscribe

---

## 🔹 5. Replay Behavior

👉 Hot publishers can be modified to replay

### 🔸 Using replay()

```java
Flux<Integer> flux = Flux.range(1, 3)
        .replay()
        .autoConnect();
```

✔ Stores past data
✔ Late subscribers get history

---

## 🔹 6. Types of Hot Behavior

| Type      | Behavior                      |
| --------- | ----------------------------- |
| multicast | Only live data                |
| replay    | Past + new data               |
| cache     | Similar to replay but simpler |

---

## 🔹 7. Real-Life Examples

| Scenario        | Type |
| --------------- | ---- |
| REST API call   | Cold |
| Database query  | Cold |
| Kafka stream    | Hot  |
| WebSocket       | Hot  |
| UI click events | Hot  |

---

## 🔹 8. Common Interview Questions

---

### ❓ Q1: Why is Flux.just() cold?

👉 Because:

* It **replays data for every subscriber**
* Execution happens again

---

### ❓ Q2: How to make a cold publisher hot?

👉 Use:

* `share()`
* `publish().refCount()`
* `Sinks.Many`

---

### ❓ Q3: Can hot publisher replay data?

👉 Yes, using:

* `replay()`
* `cache()`

---

### ❓ Q4: Which is default in Reactor?

👉 **Cold publisher**

---

## 🔹 9. Internal Behavior (Important Insight)

### Cold:

```text
Subscriber 1 → new execution
Subscriber 2 → new execution
```

### Hot:

```text
Source → shared stream → all subscribers
```

---

## 🔹 10. Simple Interview Example

```java
Flux<Long> flux = Flux.interval(Duration.ofSeconds(1)).share();

flux.subscribe(v -> System.out.println("Sub1: " + v));

Thread.sleep(2000);

flux.subscribe(v -> System.out.println("Sub2: " + v));
```

👉 Output:

```text
Sub1: 0
Sub1: 1
Sub2: 2   // missed 0 and 1
Sub1: 2
```

---

## 🔹 11. Hot + Replay Example

```java
Flux<Long> flux = Flux.interval(Duration.ofSeconds(1))
        .replay(2)
        .autoConnect();
```

✔ New subscriber gets last 2 values

---

## 🔹 12. Quick Mental Model

```text
Cold → each subscriber gets full data
Hot  → subscribers share live data
```

---

## 🔥 Final Interview Summary (1-liner)

👉
**"Cold publishers create a new data stream per subscriber, while hot publishers share a single stream across subscribers, often used for real-time event streaming."**

---

## 🚀 Pro Tip (Important for you)

In interviews always say:

👉

* Cold = **lazy execution**
* Hot = **eager/shared execution**

👉 Bonus line:

> "Hot publishers are useful in event-driven systems, while cold publishers are ideal for request-response flows."

---

If you want next level:
👉 I can give **tricky interview scenarios (race conditions, share vs replay confusion)**
👉 or **map this with Kafka/WebFlux real-world architecture**
