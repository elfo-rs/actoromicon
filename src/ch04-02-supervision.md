# Supervision

## Startup

Once the system is started, the `system.init` actor starts all actors marked as `entrypoint()` in the topology. Usually, it's only the `system.configurers` group. Entry points must have empty configs.

Then, `system.configurers` loads the config file and sends `UpdateConfig` to all actor groups. This message is usually used to start any remaining actors in the system. Thus, you need to define routing for `UpdateConfig` with either `Outcome::Unicast` or `Outcome::Multicast` for actor keys you want to start on startup.

![](assets/startup.drawio.svg)

Note that the actor system terminates immediately if the config file is invalid at startup (`elfo::start()` returns an error).

## Reconfiguration

Reconfiguration can be caused in several cases:
* The configurer receives `ReloadConfigs` or `TryRealodConfigs`.
* The process receives `SIGHUP`, it's an equivalent to receiving `ReloadConfigs::default()`.
* The process receives `SIGUSR2`, it's an equivalent to receiving `ReloadConfigs::default().with_force(true)`.

What's the difference between default behavior and `with_force(true)` one? By default, group's up-to-date configs aren't sent across the system. In the force mode, all configs are updated, even unchanged ones.

The reconfiguration process consists of several stages:
![](assets/reconfiguration.drawio.svg)

1. Config validation: the configurer sends the `ValidateConfig` request to all groups and waits for responses. If all groups respond `Ok(_)` or discard the request (which is the default behaviour), the config is considered valid and the configurer proceeds to the next step.
2. Config update: the configurer sends `UpdateConfig` to all groups. Note that it is sent as a regular message, rather than request, making config update asynchronous. System does not wait for all configs to apply before finishing this stage and it is possible for an actor to fail while applying the new config. Despite actor failures, any further actors spawned (or restarted) will have the new config available for them.

More information about configs and the reconfiguration process is available on [the corresponding page][configuration].

## Restart
### Restart policies
Three restart policies are implemented now:
* `RestartPolicy::never` (by default) — actors are never restarted. However, they can be started again on incoming messages.
* `RestartPolicy::on_failure` — actors are restarted only after failures (the `exec` function returned `Err` or panicked).
* `RestartPolicy::always` — actors are restarted after termination (the `exec` function returned `Ok` or `()`) and failures.

If the actor is scheduled to be restarted, incoming messages cannot spawn another actor for his key.


The restart policy can be chosen while creating an `ActorGroup`. This is the default restart policy for the group, which will be used if not overridden. 
```rust
use elfo::{RestartPolicy, RestartParams, ActorGroup};

ActorGroup::new().restart_policy(RestartPolicy::always(RestartParams::new(...)))
ActorGroup::new().restart_policy(RestartPolicy::on_failure(RestartParams::new(...)))
ActorGroup::new().restart_policy(RestartPolicy::never())
```

The group restart policy can be overridden separately for each actor group in the configuration:
```toml
[group_a.system.restart_policy]
when = "Never" # Override group policy on `config_update` with `RestartPolicy::never()`.

[group_b.system.restart_policy]
# Override the group policy on `config_update` with `RestartPolicy::always(...)`.
when = "Always" # or "OnFailure".
# Configuration details for the RestartParams of the backoff strategy in the restart policy. 
# The parameters are described further in the chapter.
min_backoff = "5s" # Required.
max_backoff = "30s" # Required.
factor = "2.0" # Optional.
max_retries = "20" # Optional.
auto_reset = "5s" # Optional.

[group_c]
# If the restart policy is not specified, the restart policy from the blueprint will be used.
```

Additionally, each actor can further override its restart policy through the actor's context:
```rust
// Override the group policy with the actor's policy, which is applicable only for this lifecycle.
ctx.set_restart_policy(RestartPolicy::...);

// Restore the group policy from the configuration (if specified), or use the default group policy.
ctx.set_restart_policy(None);
```

### Repetitive restarts
Repetitive restarts are limited by an exponential backoff strategy. The delay for each subsequent retry is calculated as follows:
```
delay = min(min_backoff * pow(factor, power), max_backoff)
power = power + 1
```
So for example, when `factor` is set to `2`, `min_backoff` is set to `4`, and `max_backoff` is set to `36`, and `power` is set to `0`,
this will result in the following sequence of delays: `[4, 8, 16, 32, 36]`.

The backoff strategy is configured by passing a `RestartParams` to one of the restart policies, either `RestartPolicy::on_failure` or `RestartPolicy::always`, upon creation.
The required parameters for `RestartParams` are as follows:
* `min_backoff` - Minimal restart time limit.
* `max_backoff` - Maximum restart time limit.

The `RestartParams` can be further configured with the following optional parameters:
* `factor` - The value to multiply the current delay with for each retry attempt. Default value is `2.0`.
* `max_retries` - The limit on retry attempts, after which the actor stops attempts to restart. The default value is `NonZeroU64::MAX`, which effectively means unlimited retries.
* `auto_reset` - The duration of an actor's lifecycle sufficient to deem the actor healthy. The default value is `min_backoff`.
After this duration elapses, the backoff strategy automatically resets. The following restart will occur **without delays** and will be considered the first retry.
Subsequent restarts will have delays calculated using the formula above, with the `power` starting from 0.

## Termination

System termination is started by the `system.init` actor in several cases:
* Received `SIGTERM` or `SIGINT` signals, Unix only.
* Received `CTRL_C_EVENT` or `CTRL_BREAK_EVENT` events, Windows only.
* Too high memory usage, Unix only. Now it's 90% (ratio of RSS to total memory) and not configurable ([elfo#60](https://github.com/elfo-rs/elfo/issues/60)).

The process consists of several stages:
![](assets/termination.drawio.svg)

1. `system.init` sends to all user-defined groups "polite" `Terminate::default()` message.
    * Groups' supervisors stop spawning new actors.
    * `Terminate` is routed as a usual message with `Outcome::Broadcast` by default.
    * For `TerminationPolicy::closing` (by default): the mailbox is closed instantly, `ctx.recv()` returns `None` after already stored messages.
    * For `TerminationPolicy::manually`: the mailbox isn't closed, `ctx.recv()` returns `Terminate` in order to handle in on your own. Use `ctx.close()` to close the mailbox.
2. If some groups haven't terminated after 30s, `system.init` sends `Terminate::closing()` message.
    * `Terminate` is routed as a usual message with `Outcome::Broadcast` by default.
    * For any policy, the mailbox is closed instantly, `ctx.recv()` returns `None` after already stored messages.
3. If some groups haven't terminated after another 15s, `system.init` stops waiting for that groups.
4. Repeat the above stages for system groups.

Timeouts above aren't configurable for now ([elfo#61](https://github.com/elfo-rs/elfo/issues/61)).

The termination policy can be chosen while creating `ActorGroup`:
```rust
use elfo::group::{TerminationPolicy, ActorGroup};

ActorGroup::new().termination_policy(TerminationPolicy::closing())
ActorGroup::new().termination_policy(TerminationPolicy::manually())
```

[configuration]: ./ch04-03-configuration.md
