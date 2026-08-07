# Best architecture pattern for sinks in production

Now the important part (Architect-level thinking 🚀)

❌ Anti-pattern (DON’T DO THIS)
// multiple threads directly writing
thread1 → sink.tryEmitNext()
thread2 → sink.tryEmitNext()
thread3 → sink.tryEmitNext()

👉 This WILL cause:

failures
retries
chaos under load
✅ Pattern 1: Single Writer Principle (BEST)

👉 Only ONE thread emits into sink

✔ How?

Use a queue:

BlockingQueue<String> queue = new LinkedBlockingQueue<>();

// Producers (multiple threads)
queue.offer(data);

// Single consumer thread
new Thread(() -> {
    while (true) {
        String data = queue.take();
        sink.tryEmitNext(data);
    }
}).start();
🧠 Why this is best?
No contention
No retries
Predictable order
High performance

👉 This is how Kafka producers, Netty, Disruptor work internally

✅ Pattern 2: Use Flux.create instead of Sink (Sometimes better)
Flux<String> flux = Flux.create(emitter -> {
    executor.submit(() -> emitter.next("data"));
});

👉 Handles concurrency better internally

✅ Pattern 3: Use parallel → then serialize
Flux.range(1, 10)
    .parallel()
    .runOn(Schedulers.parallel())
    .map(i -> "data " + i)
    .sequential()   // 🔥 back to single thread
    .subscribe(sink::tryEmitNext);

✔ Safe
✔ Reactive way

✅ Pattern 4: Event Bus Wrapper (Clean Architecture)

Create a wrapper:

class EventBus {
    private final Sinks.Many<String> sink =
        Sinks.many().multicast().onBackpressureBuffer();

    public void publish(String data) {
        sink.emitNext(data, Sinks.EmitFailureHandler.FAIL_FAST);
    }

    public Flux<String> listen() {
        return sink.asFlux();
    }
}

👉 Then:

Hide sink
Control emission centrally
Q4: Brutally honest production advice

👉 If you are building microservices:

DO:
Use Single Writer Pattern
Hide sinks behind service layer
Prefer Kafka / messaging system for scaling
DON’T:
Let random threads emit directly
Ignore EmitResult
Assume Reactor handles everything
Final mental model (must remember)

👉 Sinks are NOT concurrent writers

Think:

Sink = single-lane road
Threads = cars

Only ONE car at a time allowed

If you want next level:
I can show you:

Kafka + Sink integration pattern (very useful for your backend)
How Spring WebFlux internally avoids this problem
