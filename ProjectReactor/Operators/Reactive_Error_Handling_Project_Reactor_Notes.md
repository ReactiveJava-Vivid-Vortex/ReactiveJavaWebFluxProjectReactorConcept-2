<a id="index"></a>
# Index / Cheat Sheet

> Click any operator below to jump directly to its detailed section.

| Operator | Handles | What it does |
|---|---|---|
| [`onErrorReturn`](#section-2) | Error | Return fallback value |
| [`onErrorResume`](#section-3) | Error | Switch to another Publisher |
| [`onErrorMap`](#section-4) | Error | Transform exception |
| [`onErrorComplete`](#section-5) | Error | Convert error into complete |
| [`onErrorContinue`](#section-6) | Element-processing error | Skip failing element and continue where supported |
| [`defaultIfEmpty`](#section-7) | Empty | Return fallback value |
| [`switchIfEmpty`](#section-8) | Empty | Switch to another Publisher |
| [`doOnError`](#section-10) | Error | Observe/log error; error continues |
| [`doFinally`](#section-11) | Complete/Error/Cancel | Execute final side effect |
| [`retry`](#section-12) | Error | Retry failed sequence |
| [`retryWhen`](#section-13) | Error | Controlled retry/backoff |
| [`timeout`](#section-14) | Slow operation | Turn timeout into error or fallback |
| [`using`](#section-15) | Resource lifecycle | Ensure resource cleanup |
| [`materialize`](#section-16) | Signals | Convert signals into data |

### Quick decision tree

```text
                    Something happened
                           |
              +------------+------------+
              |                         |
            EMPTY                     ERROR
              |                         |
       +------+-------+          +------+----------+
       |              |          |      |     |    |
 defaultIfEmpty  switchIfEmpty  return resume map retry
                                  |      |     |    |
                               value  publisher exception
```

### One-line memory trick

```text
EMPTY  → defaultIfEmpty / switchIfEmpty
ERROR  → onErrorReturn / onErrorResume / onErrorMap
LOG    → doOnError
CLEAN  → doFinally
RETRY  → retry / retryWhen
SLOW   → timeout
SKIP   → onErrorContinue (use carefully)
```

---

# 1. Quick mental model

In Reactor, an error is a **terminal signal**.

```text
onNext → onNext → onError
                    ↓
              error handling
```

Once `onError` is emitted, the original sequence terminates. Operators such as `onErrorReturn`, `onErrorResume`, `onErrorMap`, etc. can intercept that error and decide what should happen next.

A useful rule:

- **Fallback value needed?** → `onErrorReturn`
- **Fallback publisher / another operation needed?** → `onErrorResume`
- **Convert one exception into another?** → `onErrorMap`
- **Ignore the error and complete?** → `onErrorComplete`
- **Log/observe an error without changing the signal?** → `doOnError`
- **Retry the failed operation?** → `retry` / `retryWhen`
- **Empty is not an error?** → `defaultIfEmpty` / `switchIfEmpty`

> These operators can usually be placed near the end of the pipeline, just before the subscriber, when you want to handle errors produced by the upstream operators.

---

[↑ Back to Index](#index)

<a id="section-2"></a>
# 2. `onErrorReturn` — return a fallback value

Use this when an error occurs and you simply want to return **one fallback value**.

### Example

```java
Mono.just("10")
    .map(Integer::parseInt)
    .onErrorReturn(-1)
    .subscribe(System.out::println);
```

If parsing fails:

```text
-1
```

### Variants

#### Any exception

```java
.onErrorReturn(-1)
```

Handles any error.

#### Specific exception type

```java
.onErrorReturn(IllegalArgumentException.class, -1)
```

Handles only `IllegalArgumentException`.

For example:

```java
Mono.just("abc")
    .map(Integer::parseInt)
    .onErrorReturn(NumberFormatException.class, -1);
```

`NumberFormatException` is an `IllegalArgumentException`, but the operator checks the exception type you specify.

#### Predicate-based

Reactor also provides a predicate-based overload:

```java
.onErrorReturn(
    ex -> ex instanceof IllegalArgumentException,
    -1
)
```

### When to use

Good when:

```text
error → one simple fallback value
```

Not good when the fallback requires another database/API call.

---

[↑ Back to Index](#index)

<a id="section-3"></a>
# 3. `onErrorResume` — switch to another Publisher

Use this when you need to perform another operation after an error.

Think:

```text
error
  ↓
run another Mono/Flux
  ↓
continue with that publisher
```

### Simple example

```java
Mono.just("abc")
    .map(Integer::parseInt)
    .onErrorResume(ex -> Mono.just(-1))
    .subscribe(System.out::println);
```

Output:

```text
-1
```

It looks similar to `onErrorReturn`, but `onErrorResume` is more powerful because the fallback is a **Publisher**.

### Real-world example

```java
userRepository.findById(id)
    .onErrorResume(ex -> cacheService.getUser(id));
```

Conceptually:

```text
DB fails
  ↓
try cache
```

You can also handle different exceptions:

```java
.onErrorResume(TimeoutException.class,
    ex -> cacheService.getUser(id))
```

### Key difference

```text
onErrorReturn → fallback VALUE
onErrorResume → fallback PUBLISHER
```

---

[↑ Back to Index](#index)

<a id="section-4"></a>
# 4. `onErrorMap` — convert one exception into another

Use this when the operation fails with one exception, but you want to expose a more meaningful/domain-specific exception.

### Example

```java
Mono.just("abc")
    .map(Integer::parseInt)
    .onErrorMap(
        NumberFormatException.class,
        ex -> new InvalidUserIdException("Invalid user id", ex)
    );
```

Conceptually:

```text
NumberFormatException
        ↓
InvalidUserIdException
```

This is especially useful in service layers:

```java
userRepository.findById(id)
    .onErrorMap(
        DataAccessException.class,
        ex -> new UserServiceException("Unable to load user", ex)
    );
```

Your global exception handler can then handle the domain-specific exception.

---

[↑ Back to Index](#index)

<a id="section-5"></a>
# 5. `onErrorComplete` — convert error into completion

Your note is correct: it simply turns an error into a `complete` signal.

### Example

```java
Flux.just(1, 2, 0, 4)
    .map(i -> 10 / i)
    .onErrorComplete()
    .subscribe(
        System.out::println,
        ex -> System.out.println("Error"),
        () -> System.out.println("Completed")
    );
```

Output is effectively:

```text
10
5
Completed
```

The `ArithmeticException` is swallowed.

### Is it useful?

It can be useful when an error should intentionally mean:

> "There is nothing more to emit; just finish."

But it should be used carefully because it can **hide failures**.

You can also restrict it:

```java
.onErrorComplete(ArithmeticException.class)
```

or:

```java
.onErrorComplete(ex -> ex instanceof ArithmeticException)
```

---

[↑ Back to Index](#index)

<a id="section-6"></a>
# 6. `onErrorContinue` — skip the failing element and continue

This one needs special attention.

`onErrorContinue` is different from most error operators because it can allow the sequence to continue after an error caused by processing an individual element.

### Example

```java
Flux.just(1, 2, 0, 4)
    .map(i -> 10 / i)
    .onErrorContinue((ex, value) ->
        System.out.println("Skipping: " + value)
    )
    .subscribe(System.out::println);
```

Conceptually:

```text
1 → 10
2 → 5
0 → ERROR → skip 0
4 → 2.5
```

The `BiConsumer` receives:

```java
(ex, value)
```

where:

- `ex` = exception
- `value` = value associated with the failure, when Reactor can provide it

### Your logging example

```java
.onErrorContinue((ex, obj) ->
    log.error("Error processing {}", obj, ex)
)
```

### Important warning

Do **not** think of `onErrorContinue` as a normal replacement for `onErrorResume`.

It has special semantics and can affect compatible upstream operators. It is generally better to use localized error handling with `onErrorResume` when possible.

For example, instead of:

```java
Flux.just(1, 2, 0, 4)
    .map(i -> 10 / i)
    .onErrorContinue(...);
```

you can often make the failure handling explicit:

```java
Flux.just(1, 2, 0, 4)
    .flatMap(i ->
        Mono.fromCallable(() -> 10 / i)
            .onErrorResume(ex -> Mono.empty())
    );
```

This makes it clear that **only that individual operation** is being skipped.

---

[↑ Back to Index](#index)

<a id="section-7"></a>
# 7. `defaultIfEmpty` — fallback for EMPTY, not ERROR

This is important:

> `defaultIfEmpty` does NOT handle exceptions.

It handles a sequence that completes without emitting anything.

### Example

```java
Mono.empty()
    .defaultIfEmpty(-1)
    .subscribe(System.out::println);
```

Output:

```text
-1
```

But:

```java
Mono.error(new RuntimeException())
    .defaultIfEmpty(-1);
```

does **not** produce `-1`.

The error still happens.

### Mental model

```text
EMPTY
 ↓
defaultIfEmpty
 ↓
fallback value
```

---

[↑ Back to Index](#index)

<a id="section-8"></a>
# 8. `switchIfEmpty` — fallback Publisher for EMPTY

`switchIfEmpty` is similar to `defaultIfEmpty`, but instead of supplying a single value, you supply another Publisher.

### Simple example

```java
Mono.empty()
    .switchIfEmpty(Mono.just("fallback"))
    .subscribe(System.out::println);
```

Output:

```text
fallback
```

### Real-world example: cache → database

This is a very common reactive pattern:

```java
cache.getUser(id)
    .switchIfEmpty(userRepository.findById(id));
```

Conceptually:

```text
Cache
  ↓
empty?
  ↓ yes
Database
```

This is one of the most useful differences:

```text
defaultIfEmpty → fallback VALUE
switchIfEmpty  → fallback PUBLISHER
```

---

[↑ Back to Index](#index)

<a id="section-9"></a>
# 9. `switchIfEmpty(Mono.error(...))` — convert EMPTY into ERROR

This is useful when "not found" should become an exception.

### Example

```java
userRepository.findById(id)
    .switchIfEmpty(
        Mono.error(new UserNotFoundException("User not found"))
    );
```

Conceptually:

```text
User exists
    ↓
return User

User doesn't exist
    ↓
Mono.empty()
    ↓
switchIfEmpty
    ↓
Mono.error(...)
    ↓
global exception handler
```

This is different from an actual database/API error.

There are two different situations:

```text
ERROR → something went wrong
EMPTY → operation succeeded but found nothing
```

---

[↑ Back to Index](#index)

<a id="section-10"></a>
# 10. `doOnError` — observe/log an error

`doOnError` does **not** handle or replace the error.

It is mainly for side effects such as:

- logging
- metrics
- tracing
- auditing

### Example

```java
userRepository.findById(id)
    .doOnError(ex ->
        log.error("Failed to find user {}", id, ex)
    );
```

The error continues downstream.

Think:

```text
Error
  ↓
doOnError → log
  ↓
Error continues
```

Compare:

```text
doOnError     → observe
onErrorReturn → recover with value
onErrorResume → recover with publisher
onErrorMap    → transform error
```

---

[↑ Back to Index](#index)

<a id="section-11"></a>
# 11. `doFinally` — run something when the sequence terminates

`doFinally` runs when the sequence terminates by:

- complete
- error
- cancellation

### Example

```java
userRepository.findById(id)
    .doFinally(signal ->
        log.info("Finished with {}", signal)
    );
```

Useful for:

- cleanup
- timing
- metrics
- releasing resources

It is different from `doOnError` because it is not only for errors.

---

[↑ Back to Index](#index)

<a id="section-12"></a>
# 12. `retry` — retry when an error occurs

Sometimes the correct response to an error is simply to try again.

### Example

```java
webClient.get()
    .retrieve()
    .bodyToMono(String.class)
    .retry(3);
```

Conceptually:

```text
request
  ↓
ERROR
  ↓
retry
  ↓
request
  ↓
ERROR
  ↓
retry
  ↓
request
```

`retry(3)` means the original operation can be retried up to 3 times after failure.

Be careful with retries for non-idempotent operations such as some POST operations.

---

[↑ Back to Index](#index)

<a id="section-13"></a>
# 13. `retryWhen` — controlled retry

`retryWhen` provides more control over retry behavior.

For example, you can use backoff:

```java
.retryWhen(
    Retry.backoff(3, Duration.ofSeconds(1))
)
```

Conceptually:

```text
failure
 ↓
wait
 ↓
retry
 ↓
failure
 ↓
wait longer
 ↓
retry
```

This is much more appropriate for transient failures such as temporary network problems.

---

[↑ Back to Index](#index)

<a id="section-14"></a>
# 14. `timeout` — turn slow operations into errors

Although not strictly an error-handler, `timeout` is commonly used with reactive error handling.

### Example

```java
webClient.get()
    .retrieve()
    .bodyToMono(String.class)
    .timeout(Duration.ofSeconds(2))
    .onErrorReturn("fallback");
```

Conceptually:

```text
API takes > 2 seconds
        ↓
TimeoutException
        ↓
fallback
```

You can also provide a fallback Publisher directly:

```java
.timeout(
    Duration.ofSeconds(2),
    Mono.just("fallback")
)
```

---

[↑ Back to Index](#index)

<a id="section-15"></a>
# 15. `using` / resource cleanup

For resources that need deterministic cleanup, Reactor provides `using`.

The general idea is:

```text
create resource
      ↓
use resource
      ↓
cleanup resource
```

Example:

```java
Flux.using(
    () -> createResource(),
    resource -> useResource(resource),
    resource -> resource.close()
);
```

This is useful when working with resources that have explicit lifecycle management.

---

[↑ Back to Index](#index)

<a id="section-16"></a>
# 16. `materialize` / `dematerialize` — treat signals as data

These are advanced operators.

Normally Reactor treats:

```text
onNext
onComplete
onError
```

as signals.

`materialize()` converts those signals into `Signal` objects so they can be processed as data.

```java
flux.materialize()
```

and:

```java
flux.dematerialize()
```

converts the signals back.

These are useful for advanced reactive processing and are **not normally needed for everyday Spring WebFlux code**.

---

[↑ Back to Index](#index)

<a id="section-17"></a>
# 17. Very important: Error handling vs Empty handling

Keep these two concepts separate.

## Error

Something went wrong:

```java
Mono.error(new RuntimeException())
```

Use:

```text
onErrorReturn
onErrorResume
onErrorMap
onErrorComplete
onErrorContinue
retry
retryWhen
doOnError
```

## Empty

The operation completed successfully but produced no value:

```java
Mono.empty()
```

Use:

```text
defaultIfEmpty
switchIfEmpty
```

This distinction is extremely important in WebFlux.

---

[↑ Back to Index](#index)

<a id="section-18"></a>
# 18. Your repository example

Suppose:

```java
userRepository.findById(id)
```

returns:

```java
Mono<User>
```

You want:

1. If user doesn't exist → `UserNotFoundException`
2. If DB has a technical failure → log it
3. If a transient failure occurs → retry
4. Let the global exception handler deal with the final exception

A possible pipeline:

```java
userRepository.findById(id)
    .switchIfEmpty(
        Mono.error(new UserNotFoundException("User not found"))
    )
    .doOnError(ex ->
        log.error("Failed to retrieve user {}", id, ex)
    )
    .retryWhen(Retry.backoff(3, Duration.ofMillis(200)));
```

The important thing is understanding what each operator is doing rather than putting every error operator into every pipeline.

---

# 1. Quick mental model

In Reactor, an error is a **terminal signal**.

```text
onNext → onNext → onError
                    ↓
              error handling
```

Once `onError` is emitted, the original sequence terminates. Operators such as `onErrorReturn`, `onErrorResume`, `onErrorMap`, etc. can intercept that error and decide what should happen next.

A useful rule:

- **Fallback value needed?** → `onErrorReturn`
- **Fallback publisher / another operation needed?** → `onErrorResume`
- **Convert one exception into another?** → `onErrorMap`
- **Ignore the error and complete?** → `onErrorComplete`
- **Log/observe an error without changing the signal?** → `doOnError`
- **Retry the failed operation?** → `retry` / `retryWhen`
- **Empty is not an error?** → `defaultIfEmpty` / `switchIfEmpty`

> These operators can usually be placed near the end of the pipeline, just before the subscriber, when you want to handle errors produced by the upstream operators.

---

[↑ Back to Index](#index)

<a id="section-2"></a>
# 2. `onErrorReturn` — return a fallback value

Use this when an error occurs and you simply want to return **one fallback value**.

### Example

```java
Mono.just("10")
    .map(Integer::parseInt)
    .onErrorReturn(-1)
    .subscribe(System.out::println);
```

If parsing fails:

```text
-1
```

### Variants

#### Any exception

```java
.onErrorReturn(-1)
```

Handles any error.

#### Specific exception type

```java
.onErrorReturn(IllegalArgumentException.class, -1)
```

Handles only `IllegalArgumentException`.

For example:

```java
Mono.just("abc")
    .map(Integer::parseInt)
    .onErrorReturn(NumberFormatException.class, -1);
```

`NumberFormatException` is an `IllegalArgumentException`, but the operator checks the exception type you specify.

#### Predicate-based

Reactor also provides a predicate-based overload:

```java
.onErrorReturn(
    ex -> ex instanceof IllegalArgumentException,
    -1
)
```

### When to use

Good when:

```text
error → one simple fallback value
```

Not good when the fallback requires another database/API call.

---

[↑ Back to Index](#index)

<a id="section-3"></a>
# 3. `onErrorResume` — switch to another Publisher

Use this when you need to perform another operation after an error.

Think:

```text
error
  ↓
run another Mono/Flux
  ↓
continue with that publisher
```

### Simple example

```java
Mono.just("abc")
    .map(Integer::parseInt)
    .onErrorResume(ex -> Mono.just(-1))
    .subscribe(System.out::println);
```

Output:

```text
-1
```

It looks similar to `onErrorReturn`, but `onErrorResume` is more powerful because the fallback is a **Publisher**.

### Real-world example

```java
userRepository.findById(id)
    .onErrorResume(ex -> cacheService.getUser(id));
```

Conceptually:

```text
DB fails
  ↓
try cache
```

You can also handle different exceptions:

```java
.onErrorResume(TimeoutException.class,
    ex -> cacheService.getUser(id))
```

### Key difference

```text
onErrorReturn → fallback VALUE
onErrorResume → fallback PUBLISHER
```

---

[↑ Back to Index](#index)

<a id="section-4"></a>
# 4. `onErrorMap` — convert one exception into another

Use this when the operation fails with one exception, but you want to expose a more meaningful/domain-specific exception.

### Example

```java
Mono.just("abc")
    .map(Integer::parseInt)
    .onErrorMap(
        NumberFormatException.class,
        ex -> new InvalidUserIdException("Invalid user id", ex)
    );
```

Conceptually:

```text
NumberFormatException
        ↓
InvalidUserIdException
```

This is especially useful in service layers:

```java
userRepository.findById(id)
    .onErrorMap(
        DataAccessException.class,
        ex -> new UserServiceException("Unable to load user", ex)
    );
```

Your global exception handler can then handle the domain-specific exception.

---

[↑ Back to Index](#index)

<a id="section-5"></a>
# 5. `onErrorComplete` — convert error into completion

Your note is correct: it simply turns an error into a `complete` signal.

### Example

```java
Flux.just(1, 2, 0, 4)
    .map(i -> 10 / i)
    .onErrorComplete()
    .subscribe(
        System.out::println,
        ex -> System.out.println("Error"),
        () -> System.out.println("Completed")
    );
```

Output is effectively:

```text
10
5
Completed
```

The `ArithmeticException` is swallowed.

### Is it useful?

It can be useful when an error should intentionally mean:

> "There is nothing more to emit; just finish."

But it should be used carefully because it can **hide failures**.

You can also restrict it:

```java
.onErrorComplete(ArithmeticException.class)
```

or:

```java
.onErrorComplete(ex -> ex instanceof ArithmeticException)
```

---

[↑ Back to Index](#index)

<a id="section-6"></a>
# 6. `onErrorContinue` — skip the failing element and continue

This one needs special attention.

`onErrorContinue` is different from most error operators because it can allow the sequence to continue after an error caused by processing an individual element.

### Example

```java
Flux.just(1, 2, 0, 4)
    .map(i -> 10 / i)
    .onErrorContinue((ex, value) ->
        System.out.println("Skipping: " + value)
    )
    .subscribe(System.out::println);
```

Conceptually:

```text
1 → 10
2 → 5
0 → ERROR → skip 0
4 → 2.5
```

The `BiConsumer` receives:

```java
(ex, value)
```

where:

- `ex` = exception
- `value` = value associated with the failure, when Reactor can provide it

### Your logging example

```java
.onErrorContinue((ex, obj) ->
    log.error("Error processing {}", obj, ex)
)
```

### Important warning

Do **not** think of `onErrorContinue` as a normal replacement for `onErrorResume`.

It has special semantics and can affect compatible upstream operators. It is generally better to use localized error handling with `onErrorResume` when possible.

For example, instead of:

```java
Flux.just(1, 2, 0, 4)
    .map(i -> 10 / i)
    .onErrorContinue(...);
```

you can often make the failure handling explicit:

```java
Flux.just(1, 2, 0, 4)
    .flatMap(i ->
        Mono.fromCallable(() -> 10 / i)
            .onErrorResume(ex -> Mono.empty())
    );
```

This makes it clear that **only that individual operation** is being skipped.

---

[↑ Back to Index](#index)

<a id="section-7"></a>
# 7. `defaultIfEmpty` — fallback for EMPTY, not ERROR

This is important:

> `defaultIfEmpty` does NOT handle exceptions.

It handles a sequence that completes without emitting anything.

### Example

```java
Mono.empty()
    .defaultIfEmpty(-1)
    .subscribe(System.out::println);
```

Output:

```text
-1
```

But:

```java
Mono.error(new RuntimeException())
    .defaultIfEmpty(-1);
```

does **not** produce `-1`.

The error still happens.

### Mental model

```text
EMPTY
 ↓
defaultIfEmpty
 ↓
fallback value
```

---

[↑ Back to Index](#index)

<a id="section-8"></a>
# 8. `switchIfEmpty` — fallback Publisher for EMPTY

`switchIfEmpty` is similar to `defaultIfEmpty`, but instead of supplying a single value, you supply another Publisher.

### Simple example

```java
Mono.empty()
    .switchIfEmpty(Mono.just("fallback"))
    .subscribe(System.out::println);
```

Output:

```text
fallback
```

### Real-world example: cache → database

This is a very common reactive pattern:

```java
cache.getUser(id)
    .switchIfEmpty(userRepository.findById(id));
```

Conceptually:

```text
Cache
  ↓
empty?
  ↓ yes
Database
```

This is one of the most useful differences:

```text
defaultIfEmpty → fallback VALUE
switchIfEmpty  → fallback PUBLISHER
```

---

[↑ Back to Index](#index)

<a id="section-9"></a>
# 9. `switchIfEmpty(Mono.error(...))` — convert EMPTY into ERROR

This is useful when "not found" should become an exception.

### Example

```java
userRepository.findById(id)
    .switchIfEmpty(
        Mono.error(new UserNotFoundException("User not found"))
    );
```

Conceptually:

```text
User exists
    ↓
return User

User doesn't exist
    ↓
Mono.empty()
    ↓
switchIfEmpty
    ↓
Mono.error(...)
    ↓
global exception handler
```

This is different from an actual database/API error.

There are two different situations:

```text
ERROR → something went wrong
EMPTY → operation succeeded but found nothing
```

---

[↑ Back to Index](#index)

<a id="section-10"></a>
# 10. `doOnError` — observe/log an error

`doOnError` does **not** handle or replace the error.

It is mainly for side effects such as:

- logging
- metrics
- tracing
- auditing

### Example

```java
userRepository.findById(id)
    .doOnError(ex ->
        log.error("Failed to find user {}", id, ex)
    );
```

The error continues downstream.

Think:

```text
Error
  ↓
doOnError → log
  ↓
Error continues
```

Compare:

```text
doOnError     → observe
onErrorReturn → recover with value
onErrorResume → recover with publisher
onErrorMap    → transform error
```

---

[↑ Back to Index](#index)

<a id="section-11"></a>
# 11. `doFinally` — run something when the sequence terminates

`doFinally` runs when the sequence terminates by:

- complete
- error
- cancellation

### Example

```java
userRepository.findById(id)
    .doFinally(signal ->
        log.info("Finished with {}", signal)
    );
```

Useful for:

- cleanup
- timing
- metrics
- releasing resources

It is different from `doOnError` because it is not only for errors.

---

[↑ Back to Index](#index)

<a id="section-12"></a>
# 12. `retry` — retry when an error occurs

Sometimes the correct response to an error is simply to try again.

### Example

```java
webClient.get()
    .retrieve()
    .bodyToMono(String.class)
    .retry(3);
```

Conceptually:

```text
request
  ↓
ERROR
  ↓
retry
  ↓
request
  ↓
ERROR
  ↓
retry
  ↓
request
```

`retry(3)` means the original operation can be retried up to 3 times after failure.

Be careful with retries for non-idempotent operations such as some POST operations.

---

[↑ Back to Index](#index)

<a id="section-13"></a>
# 13. `retryWhen` — controlled retry

`retryWhen` provides more control over retry behavior.

For example, you can use backoff:

```java
.retryWhen(
    Retry.backoff(3, Duration.ofSeconds(1))
)
```

Conceptually:

```text
failure
 ↓
wait
 ↓
retry
 ↓
failure
 ↓
wait longer
 ↓
retry
```

This is much more appropriate for transient failures such as temporary network problems.

---

[↑ Back to Index](#index)

<a id="section-14"></a>
# 14. `timeout` — turn slow operations into errors

Although not strictly an error-handler, `timeout` is commonly used with reactive error handling.

### Example

```java
webClient.get()
    .retrieve()
    .bodyToMono(String.class)
    .timeout(Duration.ofSeconds(2))
    .onErrorReturn("fallback");
```

Conceptually:

```text
API takes > 2 seconds
        ↓
TimeoutException
        ↓
fallback
```

You can also provide a fallback Publisher directly:

```java
.timeout(
    Duration.ofSeconds(2),
    Mono.just("fallback")
)
```

---

[↑ Back to Index](#index)

<a id="section-15"></a>
# 15. `using` / resource cleanup

For resources that need deterministic cleanup, Reactor provides `using`.

The general idea is:

```text
create resource
      ↓
use resource
      ↓
cleanup resource
```

Example:

```java
Flux.using(
    () -> createResource(),
    resource -> useResource(resource),
    resource -> resource.close()
);
```

This is useful when working with resources that have explicit lifecycle management.

---

[↑ Back to Index](#index)

<a id="section-16"></a>
# 16. `materialize` / `dematerialize` — treat signals as data

These are advanced operators.

Normally Reactor treats:

```text
onNext
onComplete
onError
```

as signals.

`materialize()` converts those signals into `Signal` objects so they can be processed as data.

```java
flux.materialize()
```

and:

```java
flux.dematerialize()
```

converts the signals back.

These are useful for advanced reactive processing and are **not normally needed for everyday Spring WebFlux code**.

---

[↑ Back to Index](#index)

<a id="section-17"></a>
# 17. Very important: Error handling vs Empty handling

Keep these two concepts separate.

## Error

Something went wrong:

```java
Mono.error(new RuntimeException())
```

Use:

```text
onErrorReturn
onErrorResume
onErrorMap
onErrorComplete
onErrorContinue
retry
retryWhen
doOnError
```

## Empty

The operation completed successfully but produced no value:

```java
Mono.empty()
```

Use:

```text
defaultIfEmpty
switchIfEmpty
```

This distinction is extremely important in WebFlux.

---

[↑ Back to Index](#index)

<a id="section-18"></a>
# 18. Your repository example

Suppose:

```java
userRepository.findById(id)
```

returns:

```java
Mono<User>
```

You want:

1. If user doesn't exist → `UserNotFoundException`
2. If DB has a technical failure → log it
3. If a transient failure occurs → retry
4. Let the global exception handler deal with the final exception

A possible pipeline:

```java
userRepository.findById(id)
    .switchIfEmpty(
        Mono.error(new UserNotFoundException("User not found"))
    )
    .doOnError(ex ->
        log.error("Failed to retrieve user {}", id, ex)
    )
    .retryWhen(Retry.backoff(3, Duration.ofMillis(200)));
```

The important thing is understanding what each operator is doing rather than putting every error operator into every pipeline.

---

<a id="index"></a>
# Index / Cheat Sheet

> Click any operator below to jump directly to its detailed section.

| Operator | Handles | What it does |
|---|---|---|
| [`onErrorReturn`](#section-2) | Error | Return fallback value |
| [`onErrorResume`](#section-3) | Error | Switch to another Publisher |
| [`onErrorMap`](#section-4) | Error | Transform exception |
| [`onErrorComplete`](#section-5) | Error | Convert error into complete |
| [`onErrorContinue`](#section-6) | Element-processing error | Skip failing element and continue where supported |
| [`defaultIfEmpty`](#section-7) | Empty | Return fallback value |
| [`switchIfEmpty`](#section-8) | Empty | Switch to another Publisher |
| [`doOnError`](#section-10) | Error | Observe/log error; error continues |
| [`doFinally`](#section-11) | Complete/Error/Cancel | Execute final side effect |
| [`retry`](#section-12) | Error | Retry failed sequence |
| [`retryWhen`](#section-13) | Error | Controlled retry/backoff |
| [`timeout`](#section-14) | Slow operation | Turn timeout into error or fallback |
| [`using`](#section-15) | Resource lifecycle | Ensure resource cleanup |
| [`materialize`](#section-16) | Signals | Convert signals into data |

### Quick decision tree

```text
                    Something happened
                           |
              +------------+------------+
              |                         |
            EMPTY                     ERROR
              |                         |
       +------+-------+          +------+----------+
       |              |          |      |     |    |
 defaultIfEmpty  switchIfEmpty  return resume map retry
                                  |      |     |    |
                               value  publisher exception
```

### One-line memory trick

```text
EMPTY  → defaultIfEmpty / switchIfEmpty
ERROR  → onErrorReturn / onErrorResume / onErrorMap
LOG    → doOnError
CLEAN  → doFinally
RETRY  → retry / retryWhen
SLOW   → timeout
SKIP   → onErrorContinue (use carefully)
```

---

[↑ Back to Index](#index)

<a id="section-20"></a>
# 20. Most important operators to remember

For day-to-day Spring WebFlux development, focus first on these:

```text
onErrorReturn
onErrorResume
onErrorMap
doOnError
switchIfEmpty
defaultIfEmpty
retryWhen
timeout
doFinally
```

And remember this simple decision tree:

```text
                    Something happened
                           |
              +------------+------------+
              |                         |
            EMPTY                     ERROR
              |                         |
       +------+-------+          +------+----------+
       |              |          |      |     |    |
 defaultIfEmpty  switchIfEmpty  return resume map retry
                                  |      |     |    |
                               value  publisher exception
```

## One-line memory trick

```text
EMPTY  → defaultIfEmpty / switchIfEmpty

ERROR  → onErrorReturn / onErrorResume / onErrorMap

LOG    → doOnError

CLEAN  → doFinally

RETRY  → retry / retryWhen

SLOW   → timeout

SKIP   → onErrorContinue (use carefully)
```

[↑ Back to Index](#index)

