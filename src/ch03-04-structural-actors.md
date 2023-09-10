# Structural Actors

Organizing actors using the functions can quickly escalate in complexity, making scalability an issue. A more streamlined approach is to employ structural actors, where all actor-related code is consolidated within a specific actor structure. Notably, `elfo` does not provide any `Actor` trait, precluding constructs like `impl Actor for Counter`. As such, the following code serves as a flexible pattern that can be adapted as required.

Complex actors require decomposition into multiple related functions, which scales worse with functional actors considered earlier. The better way to organize actors is to use so-called structural actors. The idea is to write all actor-related code in some specific actor structure. `elfo` has no any `Actor` trait, so it's impossible to see `impl Actor for Counter` or something like this. Thus, the following code is some sort of pattern and can be modified if needed.

To illustrate, let's reconfigure the counter actor to adopt a structural style. The protocol code remains unchanged:
```rust
#[message]
pub struct Increment {
    pub delta: u32,
}

#[message(ret = u32)]
pub struct GetValue;
```

Here's how the counter's code transitions to this new approach:
```rust
use elfo::prelude::*;

// This constitutes the sole public API of the actor.
// It gets invoked as `counters.mount(counter::new())` in services.
pub fn new() -> Blueprint {
    ActorGroup::new().exec(|mut ctx| Counter::new(ctx).main())
}

struct Counter {
    ctx: Context,
    value: u32,
}

impl Counter {
    fn new(ctx: Context) -> Self {
        Self {
            ctx,
            value: 0,
        }
    }

    async fn main(mut self) {
        while let Some(envelope) = self.ctx.recv().await {
            msg!(match envelope {
                // More elaborate handling code can be delegated to methods.
                // These methods can easily be async.
                msg @ Increment => self.on_increment(msg),

                // Simpler code, however, can still be processed directly here.
                (GetValue, token) => {
                    self.ctx.respond(token, self.value);
                },
            })
        }
    }

    fn on_increment(&mut self, msg: Increment) {
        self.value += msg.delta;
    }
}
```

This refactoring can significantly enhances readability in many cases, making it an advantageous choice for complex actors.

Furthermore, sources can be encapsulated within the actor structure, facilitating operation from methods. To illustrate, we can introduce a feature that outputs the current value when the counter stabilizes (i.e. remains unchanged for a period). Here's how:
```rust
use std::{time::Duration, mem};
use elfo::time::Delay;

struct Counter {
    ctx: Context,
    stable_delay: Delay<StableTick>,
    value: u32,
}

#[message]
struct StableTick;

impl Counter {
    fn new(mut ctx: Context) -> Self {
        let delay = Delay::new(Duration::from_secs(1), StableTick);

        Self {
            value: 0,
            stable_delay: ctx.attach(delay),
            ctx,
        }
    }

    async fn main(mut self) {
        // ...
                StableTick => self.on_stable(),
        // ...
    }

    fn on_increment(&mut self, msg: Increment) {
        self.value += msg.delta;
        self.stable_delay.reset();
    }

    fn on_stable(&self) {
        tracing::info!("counter is stable");
    }
}
```
