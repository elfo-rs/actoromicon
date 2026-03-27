# Observability

In production systems, knowing that the code runs is not enough, you need to know *how* it runs. Observability answers questions like: how actors are processing requests? What happened in the system a minute before the outage? Without good tooling, these questions can only be answered by adding more logging or metrics after the fact, redeploying, and waiting for the problem to happen again.

In a distributed actor system these challenges are amplified. Requests hop across actors and nodes, work is performed concurrently, and a single logical operation may touch dozens of components. Retrofitting observability onto such a system rarely works well: developers forget to propagate context, log lines lack the identity of the actor that produced them, rate limits have to be re-implemented in every project and so on.

This is why elfo treats observability as a first-class concern implemented at the framework level. Things like logging, telemetry, and message dumping are built-in batteries that work correctly out of the box and, critically, they work *implicitly*. Actor code does not need to pass meta to logs or metrics handles, or a trace context through every function call. The framework provides that context automatically.

## Scope

The mechanism behind this implicit context is [`Scope`].

Every task running inside elfo has a `Scope` installed as a task-local value. Actors get one automatically; by default, there is exactly one `Scope` per running actor task. It carries everything the observability machinery needs: actor location, identity, current trace, limits for logging and dumping and so on.

When a `tracing::info!` call fires inside an actor, the logging layer reads the current task's `Scope` to decide whether the event passes the configured level, whether it exceeds the rate limit, and which actor group and key to attach to the log line, all without the actor knowing any of this is happening.

## Scope propagation

Sometimes you need to run code outside the normal actor task, for instance, in `tokio::task::spawn_blocking`. If you use elfo's own `elfo::task::spawn_blocking`, the scope is propagated automatically:

```rust
// The scope of the current actor is automatically installed on the blocking thread.
// Logs, metrics, and dumps inside the closure are attributed to this actor.
elfo::task::spawn_blocking(|| {
    tracing::info!("running on a blocking thread, still attributed correctly");
    do_heavy_work()
}).await?;
```

If you need to spawn a task manually, for example using a third-party executor or a raw `tokio::task::spawn_blocking`, you can capture the current scope with `elfo::scope::expose()` and re-install it with `Scope::sync_within` (for blocking code) or `Scope::within` (for async code):

```rust
let scope = elfo::scope::expose(); // clone the current scope out of TLS

tokio::task::spawn_blocking(move || {
    scope.sync_within(|| {
        // Everything here runs with the actor's scope installed.
        tracing::info!("still attributed to the right actor");
        do_heavy_work()
    })
}).await?;
```

`scope::expose()` panics if called outside the actor system. Use `scope::try_expose()` in code that may run in both contexts.

[`Scope`]: https://docs.rs/elfo/0.2.0-alpha.21/elfo/scope/struct.Scope.html
