



Your question is almost perfect. A slightly more natural version would be:

> **Explain Project Reactor's `fromSupplier()`, `fromRunnable()`, `fromCallable()`, and `fromFuture()` in simple terms.**

---

# Q. Explain Project Reactor's `fromSupplier()`, `fromRunnable()`, `fromCallable()`, and `fromFuture()` in simple terms.

## First understand why these methods exist

Project Reactor is all about wrapping **existing code** into a `Mono` or `Flux`.

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

They are simply **adapters**.

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
