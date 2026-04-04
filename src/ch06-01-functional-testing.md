# Functional Testing

When testing actors, we are rarely interested in their internal state directly. What matters is observable behavior: given a sequence of incoming messages, what messages does the actor produce? This is functional testing at the scale of a single actor group.

These tests exercise the actor through its public protocol, exactly as any other actor in the system would, thus they belong in `tests/` rather than `src/`. That keeps them separate from unit tests for internal helpers, which live in `src/`.

## Setup

Add the `test-util` feature to `elfo` in `Cargo.toml`:

```toml
[dev-dependencies]
elfo = { version = "0.2.0-alpha.21", features = ["test-util"] }
tokio = { version = "1", features = ["rt", "macros"] }
```

**Note:** `test-util` exposes `elfo::test`, additional constructors for messages, and `tokio/test-util` which let control time in tests. It is intended only for tests; the feature warning printed at startup reminds you not to enable it in production builds.

## A minimal example

**Note**: the full code for this example is available in [the elfo repository][full code].

Consider a simple summator actor that accumulates values and responds to summary requests:

```rust,ignore
#[message]
pub struct Increment;

#[message]
#[derive(PartialEq)]
pub struct Added(u32);

#[message(ret = u32)]
pub struct Summarize;

#[derive(Debug, Deserialize)]
pub struct Config {
    step: u32,
}

async fn summator(mut ctx: Context<Config>) {
    let mut sum = 0;

    while let Some(envelope) = ctx.recv().await {
        msg!(match envelope {
            Increment => {
                let step = ctx.config().step;
                sum += step;
                let _ = ctx.send(Added(step)).await;
            }
            (Summarize, token) => {
                ctx.respond(token, sum);
            }
        })
    }
}

pub fn new() -> Blueprint {
    ActorGroup::new().config::<Config>().exec(summator)
}
```

The corresponding functional smoke test uses `elfo::test::proxy(blueprint, config)` to start the actor group in an isolated topology and get a dedicated handle to communicate with the actor group:

```rust,ignore
#[tokio::test]
async fn it_works() {
    // Note: `RUST_LOG=elfo` can be provided to see all messages in failed cases.

    // Define a config (usually using `toml!` or `json!`).
    let config = toml::toml! {
        step = 20
    };

    // ... or provide the default one.
    let _config = elfo::config::AnyConfig::default();

    // Wrap the actor group to take control over it.
    let mut proxy = elfo::test::proxy(summators::new(), config).await;

    // How to send messages to the group.
    proxy.send(Increment).await;
    proxy.send(Increment).await;

    // How to check actors' output.
    assert_msg!(proxy.recv().await, Added(15u32..=35)); // Note: rhs is a pattern.
    assert_msg_eq!(proxy.recv().await, Added(20));

    // How to check request-response.
    assert_eq!(proxy.request(Summarize).await, 40);
}
```

## Asserting messages

`assert_msg!` and `assert_msg_eq!` extract a typed message from an `Envelope` and check it, panicking with a clear diagnostic if the check fails.

```rust,ignore
// Pattern matching — ranges, wildcards, and guards are all valid.
assert_msg!(proxy.recv().await, Added(15u32..=35));

// Exact equality — requires PartialEq on the message type.
assert_msg_eq!(proxy.recv().await, Added(20));
```

The key difference: `assert_msg!` takes a Rust **pattern** (so ranges, nested destructuring, and `if` guards work), while `assert_msg_eq!` takes a **value** and uses `assert_eq!` under the hood.

See also `elfo::test::extract_message`, `elfo::test::extract_request` to extract messages from envelopes to further check them with custom logic.

## Synchronization

`proxy.sync().await` tries to ensure the actor has processed all messages sent so far. It's not needed for tests using `.await` communication (e.g. `recv`, `send`), but it helps when asserting the *absence* of output with `try_recv()`:

```rust,ignore
proxy.send(Increment).await;
proxy.sync().await;
// It's true if the actor sent nothing.
assert!(proxy.try_recv().await.is_none());
```

## Timeouts

`proxy.recv().await` panics if no message arrives within the timeout (default: 150 ms). Importantly, this timer runs on the real wall clock, isolated from tokio's mocked time, so `tokio::time::pause()` does not stall `recv`. The timeout can be changed with `proxy.set_recv_timeout(duration)`, but this is rarely needed: tests that use mocked time typically don't need to tune it either, since the real clock keeps ticking regardless.

## Subproxies

Actors can use `send_to` or `request_to` with a specific address rather than the [routing] system. To test those paths, create a subproxy:

```rust,ignore
#[tokio::test]
async fn it_uses_subproxies() {
    let config = toml::toml! { step = 20 };
    let mut proxy = elfo::test::proxy(summators(), config).await;

    let mut subproxy = proxy.subproxy().await;
    assert_eq!(subproxy.request(Summarize).await, 0);
    assert!(proxy.try_recv().await.is_none());

    // `send(..)` and `request(..)` always send messages to the original proxy.
    subproxy.send(Increment).await;
    assert!(subproxy.try_recv().await.is_none());
    assert_msg_eq!(proxy.recv().await, Added(20));
}
```

The subproxy has its own address, so the actor can address replies back to it specifically. Messages sent through `subproxy.send()` still go to the original actor; only the *return address* differs.

## Reconfiguration

To test how an actor reacts to config changes, send an `UpdateConfig` message carrying a new `AnyConfig`. The actor receives `ConfigUpdated` as usual, and `ctx.config()` returns the new values from that point on. See the [reconfiguration] section for how this works at the system level.

```rust,ignore
use elfo::{config::AnyConfig, messages::UpdateConfig};

let new_config = AnyConfig::deserialize(toml::toml! { step = 42 }).unwrap();
proxy.send(UpdateConfig::new(new_config)).await;
```

`UpdateConfig` can also be used as a request. The response is `Ok(())` on success or `Err(ConfigRejected)` if the new config fails to deserialize into the actor's `Config` type:

```rust,ignore
let bad_config = AnyConfig::deserialize(toml::toml! { step = -1 }).unwrap();
assert!(proxy.request(UpdateConfig::new(bad_config)).await.is_err());
// Config unchanged after rejection.
```

See `ValidateConfig` for testing config validation without applying the new config.

## Termination

To test graceful shutdown, send `Terminate::default()` and then wait for the actor to finish with `proxy.finished().await`. Any messages the actor sends during teardown can still be received after `finished()` returns, since the proxy mailbox remains open:

```rust,ignore
use elfo::messages::Terminate;

proxy.send(Terminate::default()).await;
proxy.finished().await;
let envelope = proxy.recv().await;
```

`Terminate::default()` respects the group's [termination policy]. When you need to force-close the mailbox regardless of the termination policy, use `Terminate::closing()` instead.

[full code]: https://github.com/elfo-rs/elfo/blob/master/examples/test.rs
[routing]: ./ch03-01-routing.md
[reconfiguration]: ./ch03-03-supervision.md#reconfiguration
[termination policy]: ./ch03-03-supervision.md#termination
