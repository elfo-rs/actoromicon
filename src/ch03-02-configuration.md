# Configuration

Each actor group can have its own configuration section in the TOML config file. The section name corresponds to the group's name as registered in the topology.

## Structure

A typical config file looks like this:

```toml
# Properties defined here are merged into every actor group's section.
[common]
system.logging.max_level = "Info"

# User-defined config fields for the group named in the topology.
[some_group_name]
some_param = 42

# The system.* sub-section is reserved for elfo's own settings.
system.telemetry.per_actor_key = true
```

The `[common]` section is special: its contents are merged into every actor group's config. Thus, it can be used to set defaults that apply to all groups, while still allowing individual groups to override them.

## User config

Actor groups declare their own config by calling `.config::<T>()` on `ActorGroup`. `T` must implement `serde::Deserialize` and `Debug`.

```rust
#[derive(Debug, Deserialize)]
struct MyConfig {
    interval: Duration,
    threshold: u64,
}

ActorGroup::new().config::<MyConfig>().exec(my_actor)
```

The deserialized value is accessed inside the actor via `ctx.config()`, which always returns the latest validated and applied version.

### Secrets

Fields that contain sensitive data (passwords, API keys, etc.) should be wrapped in `elfo::config::Secret<T>`. This suppresses the field's value in logs and dumps while keeping full access to the underlying value in code and allowing it to be sent over network.

```rust
use elfo::config::Secret;

#[derive(Debug, Deserialize)]
struct Config {
    password: Secret<String>,
}
```

## Handling config updates

When the config file is changed and the system reloads it (e.g. on `SIGHUP`), actors receive the `UpdateConfig` message, which silently updates the value returned by `ctx.config()` and transformed into the `ConfigUpdated` message. This allows actors to react to the new config values without needing to restart the whole system:

```rust
use elfo::{
    prelude::*,
    messages::ConfigUpdated,
    time::Interval,
};

async fn my_actor(mut ctx: Context<MyConfig>) {
    let interval = ctx.attach(Interval::new(Tick));
    interval.start(ctx.config().interval);

    while let Some(envelope) = ctx.recv().await {
        msg!(match envelope {
            ConfigUpdated => {
                // React to the new config values.
                interval.set_period(ctx.config().interval);
            }
            Tick => { /* ... */ }
        });
    }
}
```

## Validating config updates

Actors have a chance to validate the incoming config on reconfiguration by receiving the `ValidateConfig` request. Any structural / type-level validation should happen at deserialization time (consider the [`validator`] crate for that). `ValidateConfig` is for checks requiring the access to actors' state. Responding `Ok(())` accepts the config; `Err(reason)` rejects the whole reload. If the message is simply discarded (no response), validation is considered passed.

```rust
use elfo::{
    messages::{ConfigUpdated, ValidateConfig},
    prelude::*,
};

msg!(match envelope {
    (ValidateConfig { config, .. }, token) => {
        let new = ctx.unpack_config(&config);
        // Perform any dynamic validation here.
        ctx.respond(token, Ok(()));
    }
})
```

Note that `ValidateConfig` is discarded by the default router. So, you should explicitly allow it in the router:

```rust
ActorGroup::new().router(MapRouter::new(|e| {
    msg!(match e {
        ValidateConfig => Outcome::GentleUnicast(MyKey),
        _ => Outcome::Default,
    })
}))
```

See the [reconfiguration] page for more details.

## System config

Every actor group's config section may contain a `system.*` sub-table that controls built-in elfo behaviour:

| Sub-section | Struct | Description |
|---|---|---|
| `system.mailbox` | [`MailboxConfig`][docs-mailbox] | Mailbox capacity |
| `system.telemetry` | [`TelemetryConfig`][docs-telemetry] | Metrics grouping |
| `system.logging` | [`LoggingConfig`][docs-logging] | Log level and rate limits |
| `system.dumping` | [`DumpingConfig`][docs-dumping] | Message dumping on/off and rate |
| `system.restart_policy` | [`RestartPolicyConfig`][docs-restart] | Configuration of [restart policies] |

## Batteries' config

Batteries (built-in actors provided by elfo) exports their own [configuration][docs-batteries].

[`validator`]: https://lib.rs/crates/validator
[reconfiguration]: ./ch03-03-supervision.html#reconfiguration
[restart policies]: ./ch03-03-supervision.html#restart-policies
[docs-mailbox]: https://docs.rs/elfo/0.2.0-alpha.20/elfo/config/system/mailbox/struct.MailboxConfig.html
[docs-telemetry]: https://docs.rs/elfo/0.2.0-alpha.20/elfo/config/system/telemetry/struct.TelemetryConfig.html
[docs-logging]: https://docs.rs/elfo/0.2.0-alpha.20/elfo/config/system/logging/struct.LoggingConfig.html
[docs-dumping]: https://docs.rs/elfo/0.2.0-alpha.20/elfo/config/system/dumping/struct.DumpingConfig.html
[docs-restart]: https://docs.rs/elfo/0.2.0-alpha.20/elfo/config/system/restart_policy/struct.RestartPolicyConfig.html
[docs-batteries]: https://docs.rs/elfo/0.2.0-alpha.20/elfo/batteries/index.html
