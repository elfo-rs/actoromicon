# Sources

In the context of the `elfo` actor system, sources serve as conduits for integrating various streams of incoming messages, such as timers, signals, futures, and streams. They allow for a seamless amalgamation of these additional streams with the messages arriving in the mailbox. Consequently, features like tracing, telemetry, and dumps are uniformly available for both regular and source-generated messages.

You can instantiate sources using dedicated constructors that correspond to different types. It's important to note that initially, these sources are inactive; they only start generating messages once they are linked to the context through the method `ctx.attach(_)`. This method also returns a handler, facilitating the management of the source. Here is how you can utilize this method:
```rust
let unattached_source = SomeSource::new();
let source_handle = ctx.attach(unattached_source);
```

If necessary, you can detach sources at any time by invoking the `handle.terminate()` method, as shown below:
```rust
source_handle.terminate();
```

Under the hood, the storage and utilization of sources are optimized significantly to support multiple sources at the same time, thanks to the [unicycle] crate. While the system supports an unlimited number of sources without offering backpressure, it's essential to moderate the usage to prevent potential out-of-memory (OOM) errors.

## Intervals

[The `Interval` source][Interval] is designed to generate messages at a defined time period.

To activate the source, employ either the `start(period)` or `start_after(delay, period)` methods, as shown below:
```rust
use elfo::time::Interval;

#[message]
struct MyTick; // adhering to best practice by using the 'Tick' suffix

ctx.attach(Interval::new(MyTick))
    .start(Duration::from_secs(42));

while let Some(envelope) = ctx.recv().await {
    msg!(match envelope {
        MyTick => { /* handling code here */ },
    });
}
```

### Adjusting the period

In instances where you need to adjust the timer's interval, possibly as a result of configuration changes, the `interval.set_period()` method comes in handy:
```rust
use elfo::{time::Interval, messages::ConfigUpdated};

#[message]
struct MyTick; // adhering to best practice by using the 'Tick' suffix

let interval = ctx.attach(Interval::new(MyTick));
interval.start(ctx.config().period);

while let Some(envelope) = ctx.recv().await {
    msg!(match envelope {
        ConfigUpdated => {
            interval.set_period(ctx.config().period);
        },
        MyTick => { /* handling code here */ },
    });
}
```

To halt the timer without detaching the interval, use `interval.stop()`. This method differs from `interval.terminate()` as it allows for the possibility to restart the timer later using `interval.start(period)` or `start_after(delay, period)` methods.

It's essential to note that calling `interval.start()` at different points can yield varied behavior compared to invoking `interval.set_period()` on an already active interval. The `interval.set_period()` method solely modifies the existing interval without resetting the time origin, contrasting with the rescheduling functions (`start_*` methods). Here's a visual representation to illustrate the differences between these two approaches:
```
set_period(10s): | 5s | 5s | 5s |  # 10s  |   10s   |
start(10s):      | 5s | 5s | 5s |  #   10s   |   10s   |
                                   #
                              called here
```

### Tracing

Every message starts a new trace, thus a new [`TraceId`][TraceId] is generated and assigned to the current scope.

## Delays

[The `Delay` source][Delay] is designed to generate one message after a specified time:
```rust
use elfo::time::Delay;

#[message]
struct MyTick; // adhering to best practice by using the 'Tick' suffix

while let Some(envelope) = ctx.recv().await {
    msg!(match envelope {
        SomeEvent => {
            ctx.attach(Delay::new(ctx.config().delay, MyTick));
        },
        MyTick => { /* handling code here */ },
    });
}
```

This source is detached automatically after emitting a message, there is no way to reschedule it. To stop delay before emitting, use the `delay.terminate()` method.

### Tracing

The emitted message continues the current trace. The reason for it is that this source is usually used for delaying specific action, so logically it's continues the current trace.

## Signals

[The `Signal` source][Signal] is designed to generate a message once a signal is received:
```rust
use elfo::signal::{Signal, SignalKind};

#[message]
struct ReloadFile;

ctx.attach(Signal::new(SignalKind::UnixHangup, ReloadFile));

while let Some(envelope) = ctx.recv().await {
    msg!(match envelope {
        ReloadFile => { /* handling code here */ },
    });
}
```


It's based on the tokio implementation, so it should be useful to read
about [caveats][tokio signal caveats].

### Tracing

Every message starts a new trace, thus a new trace id is generated and assigned to the current scope.

## Streams

[The `Stream` source][Stream] is designed to wrap existing futures/streams of messages. Items can be either any instance of [`Message`][Message] or `Result<impl Message, impl Message>`.

Once stream is exhausted, it's detached automatically.

### Futures

Utilize `Stream::once()` when implementing subtasks such as initiating a background request:
```rust
use elfo::stream::Stream;

#[message]
struct DataFetched(u32);

#[message]
struct FetchDataFailed(String);

async fn fetch_data() -> Result<DataFetched, FetchDataFailed> {
    // ... implementation details ...
}

while let Some(envelope) = ctx.recv().await {
    msg!(match envelope {
        SomeEvent => {
            ctx.attach(Stream::once(fetch_data()));
        },
        DataFetched => { /* handling code here */ },
        FetchDataFailed => { /* error handling code here */ },
    });
}
```

### futures::Stream

`Stream::from_futures03` is used to wrap existing `futures::Stream`:
```rust
use elfo::stream::Stream;

#[message]
struct MyItem(u32);

let stream = futures::stream::iter(vec![MyItem(0), MyItem(1)]);
ctx.attach(Stream::from_futures03(stream));

while let Some(envelope) = ctx.recv().await {
    msg!(match envelope {
        MyItem => { /* handling code here */ },
    });
}
```

To produce messages of different types from the stream, it's possible to cast specific messages into `AnyMessage` (undocumented for now):
```rust
futures::stream::iter(vec![MyItem(0).upcast(), AnotherItem.upcast()])
```

### Generators

`Stream::generate` is an alternative to the [async-stream] crate, offering the same functionality without the need for macros, thereby being formatted by rustfmt:
```rust
use elfo::stream::Stream;

#[message]
struct SomeMessage(u32);

#[message]
struct AnotherMessage;

ctx.attach(Stream::generate(|mut e| async move {
    e.emit(SomeMessage(42)).await;
    e.emit(AnotherMessage).await;
}));

while let Some(envelope) = ctx.recv().await {
    msg!(match envelope {
        SomeMessage(no) | AnotherMessage => { /* handling code here */ },
    });
}
```

### Tracing
The trace handling varies depending upon the method used to create the stream:
* For `Stream::from_futures03()`: each message initiates a new trace.
* For `Stream::once()` and `Stream::generate()`: the existing trace is continued.

To override the current trace, leverage `scope::set_trace_id()` at any time.

[unicycle]: https://docs.rs/unicycle
[async-stream]: https://docs.rs/async-stream
[Interval]: https://docs.rs/elfo/0.2.0-alpha.8/elfo/time/struct.Interval.html
[Delay]: https://docs.rs/elfo/0.2.0-alpha.8/elfo/time/struct.Delay.html
[Signal]: https://docs.rs/elfo/0.2.0-alpha.8/elfo/signal/struct.Signal.html
[Stream]: https://docs.rs/elfo/0.2.0-alpha.8/elfo/stream/struct.Stream.html
[Message]: https://docs.rs/elfo/0.2.0-alpha.8/elfo/trait.Message.html
[tokio signal caveats]: https://docs.rs/tokio/latest/tokio/signal/unix/struct.Signal.html
[TraceId]: ./ch05-04-tracing.html#traceid
