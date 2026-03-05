# Multiple Runtimes

By default, all actors in an elfo system run on a single shared Tokio runtime (the one that drives `elfo::init::start`). This is usually fine, but sometimes it's desirable to isolate different actor groups or even individual actors onto their own dedicated runtimes. This is especially useful when some groups are latency-sensitive and should not be affected by other groups.

## Registering runtimes

Dedicated runtimes are registered on the `Topology` via `add_dedicated_rt`. Each call takes a filter predicate and a `tokio::runtime::Handle`:

```rust
use tokio::runtime as rt;

let producers_rt = rt::Builder::new_multi_thread()
    .worker_threads(2)
    .build()
    .unwrap();

let workers_rt = rt::Builder::new_multi_thread()
    .worker_threads(3)
    .build()
    .unwrap();

let topology = Topology::empty();

// Filters are checked in order; first match wins.
// Actors that don't match any filter run on the default runtime.
topology.add_dedicated_rt(
    |meta| meta.group == "producers",
    producers_rt.handle().clone(),
);
topology.add_dedicated_rt(
    |meta| meta.group == "workers",
    workers_rt.handle().clone(),
);
```

The filter receives `ActorMeta`, so it can be used to filter based on actor's key or group's name. Filters are evaluated in registration order and the first match wins. Actors whose group matches no filter are spawned on the default runtime (the one that drives `elfo::init::start`).

## Thread accounting

When building a system with multiple runtimes, it is useful to name all threads to distinguish them in logs, profilers and so on:

```rust
fn start_runtime(name: &str, workers: usize) -> rt::Runtime {
    use std::sync::atomic::{AtomicUsize, Ordering};

    let name = name.to_string();
    let worker_idx = AtomicUsize::new(0);

    rt::Builder::new_multi_thread()
        .worker_threads(workers)
        .enable_all()
        .thread_name_fn(move || {
            let idx = worker_idx.fetch_add(1, Ordering::SeqCst);
            format!("{name}#{idx}")
        })
        .on_thread_start(|| {
            // A good place to configure thread affinity and set scheduler
            // policy, e.g. `SCHED_FIFO`.
            //
            // Count threads in order to distinguish between workers and
            // blocking ones: first `workers` threads are workers, the rest are
            // blocking threads.
        })
        .build()
        .unwrap()
}
```

With this setup the first worker thread of the `workers` runtime will be named `workers#0`, the second `workers#1`, and so on.

Run `ps -T -p $(pidof multi_runtime)` to ensure all runtimes are started:

```text
    PID    SPID TTY          TIME CMD
2163285 2163285 pts/1    00:00:00 multi_runtime
2163285 2163319 pts/1    00:00:00 producers#0
2163285 2163320 pts/1    00:00:00 workers#0
2163285 2163321 pts/1    00:00:00 workers#1
2163285 2163322 pts/1    00:00:00 workers#2
2163285 2163323 pts/1    00:00:00 default#0
```

When actors are started, their initial thread is printed in the log message:

```text
2026-02-24 07:50:28.286933519  INFO [7446157725601366017] producers/_ - started	addr=2/129338070436	thread=producers#0
```

Note: actors can be rescheduled to a different thread within the same runtime by tokio's scheduler, but they never migrate between runtimes.

## Runtime lifecycle

The dedicated runtimes must be kept alive for the duration of the system. Because `elfo::init::start` is an async function, it must be driven by some runtime — this is the *default* runtime. The dedicated runtimes are created beforehand and shut down after the system terminates:

```rust
fn main() {
    let (topology, runtimes) = topology();

    start_runtime("default", 1).block_on(elfo::init::start(topology));

    for rt in runtimes {
        rt.shutdown_background();
    }
}
```

The dedicated runtimes are only referenced by their handles inside the topology; owning the `Runtime` values outside and dropping them after `start` returns is sufficient to ensure clean shutdown.

## Full example

Check [this example](https://github.com/elfo-rs/elfo/blob/master/examples/multi_runtime.rs) for a complete code sample.
