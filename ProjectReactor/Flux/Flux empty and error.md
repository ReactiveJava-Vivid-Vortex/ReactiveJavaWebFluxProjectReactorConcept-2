# Q1. What is `Flux.empty()`?

### Simple definition

`Flux.empty()` creates a **Flux that emits nothing** and **completes immediately**.

Think of it like an **empty delivery box**.

- 📦 A normal Flux delivers items.
- 📦 `Flux.empty()` delivers **no items**.
- ✅ It simply says, "I'm done."

---

## Real-life analogy

Imagine you order books from a library.

### Normal Flux

```
Library → Book1 → Book2 → Book3 → Finished
```

---

### Flux.empty()

```
Library → "Sorry, no books available." → Finished
```

No books are sent.

No error occurs.

The request simply completes.

---

## Example

```java
Flux<String> flux = Flux.empty();

flux.subscribe(
    System.out::println,
    System.err::println,
    () -> System.out.println("Completed")
);
```

Output

```
Completed
```

Notice

- No `onNext()`
- No values
- Only `onComplete()`

---

## Event sequence

```
Subscriber

    |
    |
    V

onSubscribe()

    |
    |
    V

onComplete()
```

No `onNext()` is called.

---

## When is it useful?

Suppose a database has no matching records.

Instead of returning `null`,

```java
return Flux.empty();
```

Example

```java
public Flux<User> getUsers() {
    return Flux.empty();
}
```

The caller still gets a valid Flux.

---

# Q2. What is `Flux.error()`?

### Simple definition

`Flux.error()` creates a Flux that **immediately fails with an exception**.

It emits **no values**.

It only sends an error signal.

---

## Real-life analogy

Imagine ordering books.

Instead of books,

the library says

> "Our database is down."

The request immediately fails.

```
Library → Error
```

No books are delivered.

---

## Example

```java
Flux<String> flux =
    Flux.error(new RuntimeException("Database Down"));

flux.subscribe(
    System.out::println,
    System.err::println,
    () -> System.out.println("Completed")
);
```

Output

```
java.lang.RuntimeException: Database Down
```

No completion happens.

---

## Event sequence

```
Subscriber

    |
    |
    V

onSubscribe()

    |
    |
    V

onError()
```

No `onNext()`.

No `onComplete()`.

---

## When is it useful?

Suppose a required configuration is missing.

```java
if (configMissing) {
    return Flux.error(new IllegalStateException("Config missing"));
}
```

Instead of throwing the exception directly, you return a Flux that represents the failure reactively.

---

# Comparison

| Feature | `Flux.empty()` | `Flux.error()` |
|---------|----------------|----------------|
| Emits values | ❌ | ❌ |
| Completes successfully | ✅ | ❌ |
| Sends error | ❌ | ✅ |
| Calls `onNext()` | ❌ | ❌ |
| Calls `onComplete()` | ✅ | ❌ |
| Calls `onError()` | ❌ | ✅ |

---

# Timeline Comparison

### `Flux.just("A", "B")`

```
Subscribe
    |
    V
A
    |
    V
B
    |
    V
Complete
```

---

### `Flux.empty()`

```
Subscribe
    |
    V
Complete
```

---

### `Flux.error(...)`

```
Subscribe
    |
    V
Error
```

---

# Common use cases

### `Flux.empty()`

When "no data" is a valid outcome.

Examples:

- Search returns no results
- User has no notifications
- Empty database query
- Feature disabled
- Optional data source

```java
if (users.isEmpty()) {
    return Flux.empty();
}
```

---

### `Flux.error()`

When the operation cannot continue.

Examples:

- Database unavailable
- Authentication failed
- Invalid input
- Configuration missing
- External service failure

```java
if (token == null) {
    return Flux.error(new IllegalArgumentException("Missing token"));
}
```

---

# How are they different from `Mono.empty()` and `Mono.error()`?

The behavior is **exactly the same**.

The only difference is what they are capable of emitting:

- `Mono` → at most **one** item
- `Flux` → **zero to many** items

So:

```java
Mono.empty();      // 0 items, then complete
Mono.error(ex);    // error immediately

Flux.empty();      // 0 items, then complete
Flux.error(ex);    // error immediately
```

---

# Easy way to remember

- **`Flux.just(...)`** → "Here are your values."
- **`Flux.empty()`** → "There are no values, but everything is OK."
- **`Flux.error()`** → "Something went wrong; here's the error."

---

You're right on spot.

# Q1. What are the use cases of `Flux.empty()`?

## Simple idea

Use `Flux.empty()` when **"no data" is a perfectly valid result**, not an error.

Think of it as saying:

> "I looked for the data. There just isn't any."

---

## Use Case 1: Search returns no results (Most common)

Suppose you're searching users by city.

```java
public Flux<User> findUsers(String city) {
    if ("Mars".equals(city)) {
        return Flux.empty();
    }

    return Flux.just(
        new User("John"),
        new User("Alice")
    );
}
```

Calling

```java
findUsers("Mars")
```

Result

```
Completed
```

No users were found, but nothing went wrong.

---

## Use Case 2: Database query returns zero rows

SQL

```sql
SELECT * FROM users WHERE age > 100;
```

No records exist.

Instead of

```java
return null;
```

Return

```java
return Flux.empty();
```

This is much safer because subscribers can continue working without worrying about `NullPointerException`.

---

## Use Case 3: User has no notifications

```java
public Flux<Notification> getNotifications(User user) {

    if (user.hasNoNotifications()) {
        return Flux.empty();
    }

    return notificationRepository.findAll(user);
}
```

No notifications is normal.

Not an error.

---

## Use Case 4: Feature disabled

Suppose email sending is disabled.

```java
if (!emailEnabled) {
    return Flux.empty();
}
```

The pipeline simply finishes.

---

## Use Case 5: Filter removes everything

```java
Flux.just(2,4,6)
    .filter(i -> i % 2 != 0);
```

Output

```
Completed
```

All values were filtered away.

Internally it behaves like an empty Flux.

---

# Q2. What are the use cases of `Flux.error()`?

## Simple idea

Use `Flux.error()` when **the operation cannot continue**.

Think of it as saying:

> "I couldn't even produce the data."

---

## Use Case 1: Invalid input (Most common)

```java
public Flux<User> findUser(String id) {

    if (id == null) {
        return Flux.error(
            new IllegalArgumentException("ID cannot be null")
        );
    }

    ...
}
```

The caller immediately receives an error.

---

## Use Case 2: Authentication failed

```java
if (!authenticated) {
    return Flux.error(
        new UnauthorizedException()
    );
}
```

No data should be emitted.

---

## Use Case 3: Database is unavailable

```java
if (!databaseConnected) {
    return Flux.error(
        new RuntimeException("Database Down")
    );
}
```

There is no point continuing.

---

## Use Case 4: External service failed

Suppose you're calling a payment service.

```java
if (paymentFailed) {
    return Flux.error(
        new PaymentException()
    );
}
```

Subscribers immediately know something went wrong.

---

## Use Case 5: Business rule violated

Example:

Age must be at least 18.

```java
if (age < 18) {
    return Flux.error(
        new IllegalArgumentException("Underage")
    );
}
```

---

# Real Spring WebFlux example

Imagine this endpoint:

```java
@GetMapping("/users")
public Flux<User> getUsers() {
    return userService.getUsers();
}
```

### Case 1: Users exist

```
John
Alice
Bob
Completed
```

---

### Case 2: No users

```
Completed
```

This is `Flux.empty()`.

HTTP response could simply be:

```
200 OK
[]
```

---

### Case 3: Database crashed

```
Error
```

This is `Flux.error()`.

Spring WebFlux typically converts it into an appropriate error response (for example, a `5xx` status unless you map it differently with exception handling).

---

# Why not just throw an exception?

Instead of

```java
throw new RuntimeException("DB Down");
```

we write

```java
return Flux.error(new RuntimeException("DB Down"));
```

### Why?

Because in Reactor, **errors are part of the stream**, just like values and completion.

A reactive stream has three possible signals:

```
onNext()
onComplete()
onError()
```

`Flux.error()` produces the `onError()` signal, allowing downstream operators to handle it reactively with operators like `onErrorResume()`, `onErrorReturn()`, or `retry()`.

---

# Rule of thumb

| Situation                                           | Use                                               |
| --------------------------------------------------- | ------------------------------------------------- |
| Data exists                                         | `Flux.just(...)` or another data-producing `Flux` |
| No data is a valid outcome                          | `Flux.empty()`                                    |
| Something went wrong and processing cannot continue | `Flux.error(...)`                                 |

---

# Easy way to remember

Imagine you order food from a restaurant:

* 🍕 **`Flux.just(pizza, drink)`** → "Here is your order."
* 📦 **`Flux.empty()`** → "You didn't order anything today." (Normal situation, nothing is wrong.)
* ❌ **`Flux.error()`** → "The kitchen caught fire. We can't fulfill your order." (An actual failure.)
