# Project Structure

Although the project structure can be arbitrary, it looks reasonable to provide some advices here:
* Use a separate crate for every actor group.
    * It speeds up a build time.
    * It helps to isolate code.
* Use separate crates for protocols, where messages are defined. Internal messages (timer ticks, inner group messages etc) are defined inside a group's crate.
* Actor crates shouldn't depend on each other. Instead, they should depend on protocol crates.
    * It helps to achieve loosely coupling.
* Prefer integration tests (`tests/`) over unit tests (`mod tests`). Unit tests are still fine for internal structures.
    * Integration tests use no implementation details and relies only on protocols.
* Store libraries (additional code potentially used by multiple actors) separately from actors' code. It's ok to depends on protocols in that code, but don't depend on actors. `libs/` in the example below.
* Store different parts of a group in dedicated files, it siplifies navigation:
    * `exec()` and message handling in `actor.rs`.
    * An inner-group router in `router.rs`.
    * A group's config in `config.rs`.
* Store topology definitions separately from actors' code. Services can share actors. Also, it's a good place for additional files like systemd units and so on. `services/` in the example below.

For instance:
```
actors/
    <actor_name_1>/
        src/
            mod.rs
            actor.rs
            config.rs
            router.rs
        tests/
            helpers.rs
            test_<name_1>.rs
            test_<name_2>.rs
        Cargo.toml
    <actor_name_2>/
protocol/
    src/
        lib.rs
    Cargo.toml
libs/
    <lib_name_1>/
    <lib_name_2>/
services/
    <service_name_1>/
    <service_name_2>/
Cargo.toml
Cargo.lock
rustfmt.toml
```
