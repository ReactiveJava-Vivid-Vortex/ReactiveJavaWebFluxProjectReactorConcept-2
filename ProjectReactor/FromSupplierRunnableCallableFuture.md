



Your question is almost perfect. A slightly more natural version would be:

> **Explain Project Reactor's `fromSupplier()`, `fromRunnable()`, `fromCallable()`, and `fromFuture()` in simple terms.**

---

# Q. Explain Project Reactor's `fromSupplier()`, `fromRunnable()`, `fromCallable()`, and `fromFuture()` in simple terms.

## First understand why these methods exist

<mark>Project Reactor is all about wrapping **existing code** into a `Mono` or `Flux`.</mark>

Imagine you already have Java code like this:

```java
String getName();
void save();
User findUser();
CompletableFuture<User> findUserAsync();
```

These methods are **not reactive**.

Project Reactor provides methods to convert them into reactive publishers.

Think of it like this:

| Existing Java Type | Reactor Wrapper |
|--------------------|-----------------|
| Supplier | `Mono.fromSupplier()` |
| Callable | `Mono.fromCallable()` |
| Runnable | `Mono.fromRunnable()` |
| CompletableFuture | `Mono.fromFuture()` |

<mark>They are simply **adapters**.</mark>

---

# 1. `Mono.fromSupplier()`

## What is a Supplier?

A Supplier is simply a function that

- takes **no input**
- returns **one value**

```java
Supplier<String> supplier = () -> "Hello";
```

Equivalent normal method

```java
String getMessage() {
    return "Hello";
}
```

---

### Reactor

```java
Mono<String> mono =
        Mono.fromSupplier(() -> "Hello");
```

Nothing runs yet.

Only when someone subscribes...

```java
mono.subscribe(System.out::println);
```

Output

```
Hello
```

---

### Think of it like a coffee machine

Supplier is like

> "Whenever someone asks, prepare a fresh coffee."

Until someone asks...

Nothing happens.

---

## Timeline

```
Create Supplier

        |
        V

Mono.fromSupplier()

        |
        V

(No execution)

        |
 subscribe()
        |
        V

Supplier executes

        |
        V

Value emitted
```

---

# When to use it?

Whenever you already have code returning a value.

Example

```java
String getUserName() {
    return "Deepak";
}

Mono<String> mono =
        Mono.fromSupplier(this::getUserName);
```

---

# 2. `Mono.fromCallable()`

Looks almost identical.

Example

```java
Mono<String> mono =
        Mono.fromCallable(() -> readFile());
```

---

## What is Callable?

Callable is

- no input
- returns one value
- **can throw checked exceptions**

```java
Callable<String> task = () -> {
    return Files.readString(path);
};
```

Notice

```java
Files.readString()
```

throws IOException.

Supplier cannot throw checked exceptions.

Callable can.

---

### Why does Reactor provide it?

Because lots of blocking Java APIs throw checked exceptions.

Example

Reading a file

```java
Files.readString(path)
```

Database

```java
statement.executeQuery();
```

Network

```java
socket.read();
```

All throw checked exceptions.

So Reactor lets you wrap them safely.

---

## Example

```java
Mono<String> mono =
        Mono.fromCallable(() ->
            Files.readString(path)
        );
```

If reading succeeds

```
onNext(data)
onComplete()
```

If IOException occurs

```
onError(IOException)
```

Reactor converts exceptions into error signals automatically.

---

# Difference

Supplier

```
Returns value
Cannot throw checked exception
```

Callable

```
Returns value
Can throw checked exception
```

---

# 3. `Mono.fromRunnable()`

Runnable is different.

Runnable returns

**nothing**

Example

```java
Runnable task = () -> {
    System.out.println("Saving...");
};
```

Equivalent

```java
void save() {
    System.out.println("Saving...");
}
```

---

Reactor

```java
Mono<Void> mono =
        Mono.fromRunnable(() -> save());
```

Subscribe

```java
mono.subscribe();
```

Output

```
Saving...
```

Notice

No value is emitted.

Only completion.

```
onComplete()
```

No

```
onNext()
```

---

## Timeline

```
Runnable

      |
      V

Execute

      |
      V

No value

      |
      V

Complete
```

---

### When to use?

Whenever you have

```java
void sendEmail();

void save();

void delete();

void log();
```

Example

```java
Mono<Void> mono =
        Mono.fromRunnable(this::save);
```

---

# 4. `Mono.fromFuture()`

Suppose you already have asynchronous Java code.

Example

```java
CompletableFuture<String> future =
        CompletableFuture.supplyAsync(() -> "Hello");
```

Reactor doesn't want you to rewrite it.

Instead

```java
Mono<String> mono =
        Mono.fromFuture(future);
```

Now Reactor waits for the future.

When future completes

```
Future

     |
     V

Hello

     |
     V

Mono emits Hello
```

---

## Example

```java
CompletableFuture<String> future =
        CompletableFuture.supplyAsync(() -> {

            Thread.sleep(1000);

            return "Done";

        });

Mono.fromFuture(future)
    .subscribe(System.out::println);
```

After one second

```
Done
```

---

# Real-world example

Suppose some library returns

```java
CompletableFuture<User>
```

but your application uses Reactor.

Simply do

```java
Mono<User> mono =
        Mono.fromFuture(userService.findAsync());
```

Now everything is reactive.

---

# Complete comparison

| Method | Java Interface | Returns Value? | Throws Checked Exception? | Emits |
|---------|---------------|----------------|---------------------------|-------|
| `fromSupplier()` | `Supplier<T>` | ✅ Yes | ❌ No | `Mono<T>` |
| `fromCallable()` | `Callable<T>` | ✅ Yes | ✅ Yes | `Mono<T>` |
| `fromRunnable()` | `Runnable` | ❌ No | ❌ No (checked exceptions not allowed) | `Mono<Void>` |
| `fromFuture()` | `CompletableFuture<T>` | ✅ Eventually | Future completes exceptionally if needed | `Mono<T>` |

---

# Simple way to remember

```
Need one value?
        │
        ├── No exception → fromSupplier()
        │
        ├── Checked exception → fromCallable()
        │
Need no value?
        │
        └── fromRunnable()

Already have a CompletableFuture?
        │
        └── fromFuture()
```

---

# A note about blocking operations

Methods like `fromSupplier()` and `fromCallable()` **do not automatically make blocking code non-blocking**. They simply defer execution until subscription. If the wrapped code performs blocking I/O (e.g., reading a file or querying a database), you should typically move it to the bounded elastic scheduler:

```java
Mono.fromCallable(() -> Files.readString(path))
    .subscribeOn(Schedulers.boundedElastic());
```

This keeps your reactive event-loop threads free while the blocking work runs on a thread pool designed for such tasks.

---

Q. So basically, are fromSupplier(), fromCallable(), fromRunnable(), and fromFuture() adapters that make blocking code non-blocking? However, to truly achieve non-blocking behavior, in addition to wrapping the code with these adapters, do we also need to use .subscribeOn(Schedulers.boundedElastic())?

# Short Answer

**Almost, but not exactly.**

These methods **do not make blocking code non-blocking.**

They **adapt existing Java APIs into Reactor types (`Mono`)**.

Whether the code is blocking or non-blocking depends on **what is inside the adapter**, not on the adapter itself.

---

# Think of it this way

Suppose you have this blocking code:

```java
String data = Files.readString(path);
```

This blocks the current thread until the file is read.

Now you wrap it:

```java
Mono<String> mono =
    Mono.fromCallable(() -> Files.readString(path));
```

Did it become non-blocking?

**No.**

It is still blocking.

The only difference is:

* Before: it executed immediately.
* Now: it executes only when someone subscribes.

---

# What does `subscribeOn()` do?

This is the missing piece.

```java
Mono.fromCallable(() -> Files.readString(path))
    .subscribeOn(Schedulers.boundedElastic());
```

Now Reactor says:

> "This work is blocking, so don't execute it on the event-loop thread. Execute it on a thread from the bounded elastic pool."

So:

* The file read is **still blocking**.
* But it blocks a **worker thread** from the bounded elastic pool instead of the event-loop thread.
* The event-loop thread remains free to serve other requests.

---

# Visual comparison

### Without `subscribeOn()`

```
Event Loop Thread

      |
      V

Files.readString()

      |
      |  (Thread blocked)
      |
      V

Continue
```

The event-loop cannot process other requests during that time.

---

### With `subscribeOn(Schedulers.boundedElastic())`

```
Event Loop

      |
      |  schedules work
      V

Bounded Elastic Thread

      |
      V

Files.readString()

      |
      | (Worker thread blocked)
      |
      V

Result

      |
      V

Event Loop continues processing other requests
```

The blocking operation hasn't disappeared—it has just been moved to a thread pool designed for blocking work.

---

# Then what is truly non-blocking?

Suppose you use a reactive database driver like R2DBC.

```java
Mono<User> user = repository.findById(id);
```

There is **no blocking call** like `Thread.sleep()` or `executeQuery()` that waits.

Instead:

```
Request sent
      |
      V

Thread is FREE

      |
      V

Database responds later

      |
      V

Reactor emits result
```

The thread never waits.

That is **true non-blocking I/O**.

Notice that in this case, you **don't need** `fromCallable()` or `boundedElastic()` because the driver is already reactive.

---

# When should you use each?

### Existing blocking API

```java
Mono.fromCallable(() -> Files.readString(path))
    .subscribeOn(Schedulers.boundedElastic());
```

✅ Correct

---

### Existing JDBC

```java
Mono.fromCallable(() -> jdbcTemplate.queryForObject(...))
    .subscribeOn(Schedulers.boundedElastic());
```

✅ Correct

---

### Existing REST client using `RestTemplate`

```java
Mono.fromCallable(() -> restTemplate.getForObject(...))
    .subscribeOn(Schedulers.boundedElastic());
```

✅ Correct

---

### Reactive API (already non-blocking)

```java
webClient.get()
         .retrieve()
         .bodyToMono(User.class);
```

❌ Don't wrap it in `fromCallable()`.

❌ Don't move it to `boundedElastic()`.

It is already non-blocking.

---

# <mark>A good mental model</mark>

Think of these adapters as **wrapping paper**, not a magic machine.

* `fromSupplier()`, `fromCallable()`, `fromRunnable()`, and `fromFuture()` **wrap** existing Java code so Reactor can work with it.
* If the wrapped code is **blocking**, it remains blocking.
* <mark>`subscribeOn(Schedulers.boundedElastic())` **moves the blocking work** to a dedicated thread pool</mark> so it doesn't block Reactor's event-loop threads.
* **True non-blocking** comes from using non-blocking libraries (such as `WebClient`, R2DBC, or reactive MongoDB drivers), not from these adapters themselves.

So the common pattern for legacy blocking APIs is:

```java
Mono.fromCallable(() -> blockingOperation())
    .subscribeOn(Schedulers.boundedElastic());
```

---

Here’s a clean note you can paste below your MD using `---` separator 👇

---

### ❓ Can we use `fromCallable`, `fromSupplier`, `fromRunnable`, `fromFuture` with **Flux** as well?

### ✅ Short Answer

Yes 👍 — but **not directly in the same way as Mono**. These are primarily designed for **Mono (single value)** use cases.

---

### 🧠 Simple Explanation

Think like this:

* **Mono = 0 or 1 value**
* **Flux = 0 to many values**

Now:

| Method           | What it produces           | Best fit |
| ---------------- | -------------------------- | -------- |
| `fromSupplier()` | 1 value                    | Mono     |
| `fromCallable()` | 1 value (or error)         | Mono     |
| `fromRunnable()` | no value (just completion) | Mono     |
| `fromFuture()`   | 1 async value              | Mono     |

👉 So naturally, all these map to **Mono**, not Flux.

---

### 🤔 Then how to use with Flux?

If you really want to use them with Flux, you can **convert Mono → Flux**:

```java
Flux<Integer> flux = Mono.fromCallable(() -> 10)
                         .flux();
```

---

### 🧩 Real Use Cases

#### 1. Single value → Flux

```java
Flux<Integer> flux = Flux.from(Mono.fromSupplier(() -> 5));
```

#### 2. Multiple values → use Flux directly

```java
Flux<Integer> flux = Flux.fromIterable(List.of(1, 2, 3));
```

---

### 🚨 Important Insight (Very Important)

👉 If your source produces:

* **ONE value → use Mono**
* **MULTIPLE values → use Flux**

Don’t force `fromCallable()` etc. into Flux unnecessarily.

---

### 🎯 Interview One-Liner

> "`fromCallable`, `fromSupplier`, `fromRunnable`, and `fromFuture` are Mono-specific factory methods because they produce at most one result. If needed in Flux, we convert Mono to Flux using `.flux()`."

---

If you want, next I can extend this with **threading + blocking handling (subscribeOn)** which is where most interview questions come from 🚀


This doesn't make `blockingOperation()` non-blocking—it makes your **reactive application behave correctly** by isolating the blocking work from the event-loop.

