



Your question is **right on spot**.

# Q: Explain `Mono` ↔ `Flux` conversion in simple terms

The easiest way to remember it is:

- **Mono = 0 or 1 item**
- **Flux = 0 to many items**

Think of them like this:

- 📦 **Mono** = A single package
- 📦📦📦 **Flux** = A stream of packages

Sometimes you need to convert one into the other.

---

# 1. Mono → Flux

Suppose you have

```java
Mono<String> mono = Mono.just("Apple");
```

It contains

```
Apple
```

Now suppose a method expects a `Flux`.

You can convert it using

```java
Flux<String> flux = mono.flux();
```

Now it behaves like

```
Apple
```

Although it is a Flux, it still emits only one item.

### Output

```
Apple
Completed
```

---

## Visual

```
Mono

Apple
  │
  ▼

mono.flux()

  │
  ▼

Flux

Apple
```

Nothing magical happens.

It simply wraps the single value into a Flux.

---

# Why would you do this?

Suppose this method accepts only Flux.

```java
processOrders(Flux<Order> orders)
```

But you have

```java
Mono<Order>
```

Convert it

```java
processOrders(orderMono.flux());
```

---

# 2. Flux → Mono

Now suppose

```java
Flux<String> flux =
    Flux.just("Apple", "Banana", "Orange");
```

Flux contains

```
Apple
Banana
Orange
```

A Mono can hold only one value.

So Reactor asks:

> Which one do you want?

There are several choices.

---

# Option 1: first()

```java
Mono<String> mono = flux.next();
```

Output

```
Apple
```

Only the first item is taken.

---

Visual

```
Flux

Apple
Banana
Orange

     │
     ▼

next()

     │
     ▼

Mono

Apple
```

---

# Option 2: collectList()

Sometimes you don't want just one item.

You want **all items inside one List**.

```java
Mono<List<String>> mono =
        flux.collectList();
```

Now Mono contains

```
[
 Apple,
 Banana,
 Orange
]
```

Notice carefully:

It is still **one object**.

That one object happens to be a List.

---

Visual

```
Flux

Apple
Banana
Orange

      │
      ▼

collectList()

      │
      ▼

Mono

[
 Apple,
 Banana,
 Orange
]
```

---

# Option 3: single()

Suppose the Flux is expected to contain exactly one item.

```java
Flux.just("Apple")
```

Convert

```java
Mono<String> mono =
        flux.single();
```

Output

```
Apple
```

Works perfectly.

---

But what if Flux has

```
Apple
Banana
```

Then

```java
flux.single()
```

throws

```
IndexOutOfBoundsException
```

because there is more than one item.

Think of it as saying

> "I expected exactly one item, but I got multiple."

---

# Option 4: singleOrEmpty()

Useful when there may be **0 or 1 item**.

```
0 item ✔
1 item ✔
2 items ❌
```

---

# Option 5: last()

Take the last item.

```java
Mono<String> mono =
        flux.last();
```

Output

```
Orange
```

---

Visual

```
Flux

Apple
Banana
Orange

     │
     ▼

last()

     │
     ▼

Mono

Orange
```

---

# 3. Mono<List<T>> → Flux<T>

This conversion is very common.

Suppose

```java
Mono<List<String>> mono =
    Mono.just(List.of("A", "B", "C"));
```

It contains

```
[A, B, C]
```

Notice

This is **one List**.

If you want individual items

```
A
B
C
```

use

```java
Flux<String> flux =
        mono.flatMapMany(Flux::fromIterable);
```

or

```java
Flux<String> flux =
        mono.flatMapIterable(list -> list);
```

Output

```
A
B
C
```

---

Visual

```
Mono

[A, B, C]

      │
      ▼

flatMapMany()

      │
      ▼

Flux

A
B
C
```

---

# 4. Flux<T> → Mono<List<T>>

The opposite conversion.

```java
Flux<String> flux =
        Flux.just("A","B","C");

Mono<List<String>> mono =
        flux.collectList();
```

Result

```
Mono

[A,B,C]
```

---

# Real-world examples

### Example 1: Database

Database returns one user.

```java
Mono<User> user
```

Your API expects

```java
Flux<User>
```

Convert

```java
user.flux();
```

---

### Example 2: File processing

Read lines

```java
Flux<String>
```

Need all lines before saving

```java
flux.collectList();
```

Result

```java
Mono<List<String>>
```

---

### Example 3: REST API

API returns

```json
[
  "A",
  "B",
  "C"
]
```

Spring converts it to

```java
Mono<List<String>>
```

Need to process one item at a time

```java
mono.flatMapMany(Flux::fromIterable)
```

Now

```
A
B
C
```

---

# Summary Table

| Conversion | Method | Result |
|------------|--------|--------|
| `Mono<T>` → `Flux<T>` | `mono.flux()` | Single value becomes a Flux |
| `Flux<T>` → `Mono<T>` (first item) | `next()` | First item only |
| `Flux<T>` → `Mono<T>` (last item) | `last()` | Last item |
| `Flux<T>` → `Mono<T>` (exactly one) | `single()` | Error if 0 or >1 items |
| `Flux<T>` → `Mono<T>` (0 or 1 item) | `singleOrEmpty()` | Error only if >1 items |
| `Flux<T>` → `Mono<List<T>>` | `collectList()` | All items collected into one `List` |
| `Mono<List<T>>` → `Flux<T>` | `flatMapMany(Flux::fromIterable)` or `flatMapIterable()` | Emits each list element individually |

---

# Rule to Remember

```
Mono (0 or 1 item)
        │
        │ flux()
        ▼
Flux (0..N items)

Flux (0..N items)
        │
        ├── next()          → first item
        ├── last()          → last item
        ├── single()        → exactly one item
        ├── singleOrEmpty() → 0 or 1 item
        └── collectList()   → Mono<List<T>>

Mono<List<T>>
        │
        └── flatMapMany(Flux::fromIterable)
                ▼
             Flux<T>
```

### Productivity tip for Reactor
A common interview question is: **"When should I return `Mono<List<T>>` vs `Flux<T>` from a Spring WebFlux API?"** Learning that distinction early will make designing reactive APIs much easier. After you're comfortable with these conversions, that's the next concept worth mastering.
