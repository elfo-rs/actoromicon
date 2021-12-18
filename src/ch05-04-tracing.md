# Tracing

TODO

## `TraceId`

This ID is [periodically monotonic][monotonicity_level] and fits great as a primary key for tracing entries.
It was accomplished by using rarely (approx. once a year) wrapping `timestamp` ID component.

`TraceId` essentially is:

\\[
\operatorname{trace\\_id} =
    \operatorname{timestamp} \dot{} 2^{38}
  + \operatorname{node\\_no} \dot{} 2^{22}
  + \operatorname{chunk\\_no} \dot{} 2^{10}
  + \operatorname{counter}
\\]

This formula's parameters were chosen carefully to leave a good stock of spare space for `TraceId`'s components inside the available set of [63-bit positive integers][domain].
You may see the general idea by looking at the bits distribution table.
From MSB to LSB:

| Bits | Description         | Range            | Source |
| ---: | ------------------- | ---------------- | ------ |
|    1 | [Reserved][domain]  | `0`              | - |
|   25 | `timestamp` in secs | `0..=33_554_431` | Clock at runtime |
|   16 | `node_no`           | `0..=65_535`     | Externally specified node (process) configuration |
|   12 | `chunk_no`          | `0..=4095`       | Some number produced at runtime |
|   10 | `counter`           | `1..=1023`       | A counter inside the chunk |

The code generating `TraceId` of course can be optimized by using bit shifts for fast multiplication on a power of two:

```
trace_id = (timestamp << 38) | (node_no << 22) | (chunk_no << 10) | counter
```

`TraceId` uses time as its [monotonicity source][monotonicity_source] so **`timestamp`** is probably the most important part of the ID.
Note that `timestamp` has pretty rough resolution — in seconds.
How long you can count seconds inside 25 bits?

\\[
\operatorname{timestamp}_{max} = \frac{2^{25} - 1}{60 \dot{} 60 \dot{} 24} = \frac{33554431}{86400} \approx 388 \text{ days}
\\]

Which is almost a year plus 23 days.
What happens when this almost-one-year term ends?
`timestamp` starts counting from 0 once again:

```
TIMESTAMP_MAX = (1 << 25) - 1 // 0x1ff_ffff
timestamp = now_s() & TIMESTAMP_MAX
```

This means that primary key is guaranteed to act as a unique identifier for one year.
To keep an order of entries between monotonic periods of `TraceId` use some fully monotonic (but not necessarily unique) field as a [sorting key](https://clickhouse.tech/docs/en/engines/table-engines/mergetree-family/mergetree/#choosing-a-primary-key-that-differs-from-the-sorting-key) for your entries (`created_at` or `seq_no` / `sequential_number`).

`TraceId` with such `timestamp` part is well optimized to produce a lot of entities worth querying for only a limited period of time: [logs][logging], [dumps][dumping] or tracing entries.
However you're able to store such entries as long as you like and use [data skipping indices][monotonicity_reasons] for quick time series queries.

To make IDs unique across several instances of your system without any synchronization between them **`node_no`** ID component should be externally specified from the system deployment configuration.

**TODO: actualize information about `thread_id`: it's not used by `elfo`, but appropriate way to generate `trace_id` in other systems that want to interact with `elfo`.**

**`thread_id`** shares bit space with `counter`. At the start of the system you should determine how much threads your node could possibly have and choose appropriate \\( \operatorname{counter}_{max} \\) according to that.

Previous components segregated entries produced by separate threads on every instance of the system at different seconds.
To create multiple unique IDs during a single second inside a single thread we use **`counter`** ID component.

Let's calculate how much records per second (\\( \operatorname{RPS}_{max} \\)) allows us to produce `counter` with the bounds we've chosen.
Assuming we have 32 threads:

\\[
\operatorname{RPS}_{max} = \frac{2^{22} - 1}{\operatorname{threads\\_count}} = \frac{2^{22} - 1}{2^{5}} \approx 2^{17} \approx 1.3 \dot{} 10^{5} \text{  } \frac{\text{records}}{\text{second}}
\\]

Which seems more than enough for the most of the applications.
Note that `counter` is limited to be at least 1 to keep the [invariant][domain]: \\( \operatorname{id} \geqslant 1 \\).
Every other component of `TraceId` could be zero.

[domain]: ./ch09-01-id-generation.md#choosing-the-domain-for-your-ids
[dumping]: ./ch05-03-dumping.html
[logging]: ./ch05-01-logging.html
[monotonicity_level]: ./ch09-01-id-generation.md#level-of-monotonicity
[monotonicity_reasons]: ./ch09-01-id-generation.md#why-monotonic-ids-are-so-great
[monotonicity_source]: ./ch09-01-id-generation.md#monotonicity-source
