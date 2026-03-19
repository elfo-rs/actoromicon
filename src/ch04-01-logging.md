# Logging

Elfo uses the [`tracing`] crate as its logging interface. Actor code emits log events with the standard `tracing` macros, so no elfo-specific API is needed.

Thus, it's possible to install and use a custom [`tracing::Subscriber`]. However, elfo provides the `elfo-logger` actor with the subscriber implementation. Because it's a normal actor, it can be configured and observed. Traditionally, `elfo-logger` is mounted at the `system.loggers` actor group.

Any events emitted anywhere are captured by the subscriber, stored in the queue of the `elfo-logger` actor, and then processed by the logger's main loop according to its configuration.

![logging schema](assets/logging.drawio.svg)

## Setting up the logger

To enable logging, call `elfo::batteries::logger::init()` before building your topology and mount the returned `Blueprint` at the `system.loggers` group:

```rust
fn topology() -> elfo::Topology {
    let topology = elfo::Topology::empty();

    // Registers the global tracing subscriber; returns a Blueprint.
    let logger = elfo::batteries::logger::init();
    // From this point, every event emitted anywhere is captured.

    let loggers = topology.local("system.loggers");
    let configurers = topology.local("system.configurers").entrypoint();

    // ... user groups ...

    loggers.mount(logger);
    configurers.mount(..);

    topology
}
```

> Note: `init()` also respects the `RUST_LOG` environment variable (via `tracing-subscriber`'s `EnvFilter`) when the `env-filter` feature is enabled, which is the default.

## Usage

Use regular macros (`tracing::info!`, `tracing::warn!` etc) directly in your actors:

```rust
while let Some(envelope) = ctx.recv().await {
    msg!(match envelope {
        MyRequest { value } => {
            tracing::info!(value, "received a request!");
        }
    });
}
```

By default, every log line written by `elfo-logger` has the following structure:

```text
<timestamp> <level> [<trace_id>] <actor_group>/<actor_key> - <message>\t<fields>
```

Example output from a running system:

```text
2026-03-05 09:19:00.228757835  INFO [7661362937482706945] system.configurers/_ - status changed	status=Normal	details=updating
2026-03-05 09:19:00.228988147  INFO [7661362937482706945] producers/_ - done	sum=45
```

## Configuration

The logging configuration is defined in the [system.logging][docs-logging] subsection:

```toml
[some_group]
system.logging.max_level = "Info"
system.logging.max_rate_per_level = 1_000
```

To configure `elfo-logger` itself, use the `[system.loggers]` section:

```toml
[system.loggers]
sink = "File"
path = "app.log"
```

See the [rustdoc][docs-logger] for more details.

> Note: all configuration is hot-reloaded: when the config file is reloaded (e.g. on `SIGHUP`), the logger reconfigures itself without restarting.

## Composing a custom subscriber

If you need to compose elfo's logger with additional layers, use `elfo::batteries::logger::new()` instead of `init()`. It returns the raw parts so you can assemble your own subscriber:

```rust
let (blueprint, scope_filter, capture_layer) = elfo::batteries::logger::new();

tracing_subscriber::registry()
    .with(my_extra_layer)
    .with(capture_layer.with_filter(scope_filter))
    .init();

loggers.mount(blueprint);
```

See the [Tokio Console] chapter for a concrete example of this pattern.

[`tracing`]: https://docs.rs/tracing
[`tracing::Subscriber`]: https://docs.rs/tracing/latest/tracing/trait.Subscriber.html
[Tokio Console]: ./ch04-05-tokio-console.md
[docs-logging]: https://docs.rs/elfo/0.2.0-alpha.20/elfo/config/system/logging/struct.LoggingConfig.html
[docs-logger]: https://docs.rs/elfo/0.2.0-alpha.20/elfo/batteries/logger/config/struct.Config.html
