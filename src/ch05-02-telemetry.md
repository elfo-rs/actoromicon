# Telemetry

## Introduction

TODO: `metrics` crate, metric types.

All metrics are provided with the `actor_group` and, optionally, `actor_key` labels. The last one is added for actor groups with enabled `system.telemetry.per_actor_key` option.

Read more information about [metric types](https://prometheus.io/docs/concepts/metric_types/).

TODO: tips, prefer `increment_gauge!` over `gauge!`

## Configuration

Telemetry can be configured separately for each actor group. Possible options and their default values:
```toml
[some_group]
system.telemetry.per_actor_group = true
system.telemetry.per_actor_key = false
```

Note that using `per_actor_key` can highly increase a number of metrics. Use it only for low cardinality groups.

TODO: `elfo-telemeter` config.

## Built-in metrics

`elfo` is shipped with a lot of metrics. All of them start with the `elfo_` prefix to avoid collisions with user defined metrics.

### Statuses

* Gauge `elfo_active_actors{status}`

    The number of active actors in the specified status.

* Gauge `elfo_restarting_actors`

    The number of actors that will be restarted after some time.

* Counter `elfo_actor_status_changes_total{status}`

    The number of transitions into the specified status.

### Messages

* Counter `elfo_sent_messages_total{message, protocol}`

    The number of sent messages.

* Summary `elfo_message_handling_time_seconds{message, protocol}`

    Spent time on handling the message, measured between `(try_)recv()` calls. Used to detect slow handlers.

* Summary `elfo_message_waiting_time_seconds`

    Elapsed time between `send()` and corresponding `recv()` calls. Usually it represents a time that a message spends in a mailbox. Used to detect places that should be sharded to reduce a total latency.

* Summary `elfo_busy_time_seconds`

    Spent time on polling a task with an actor. More precisely, the time for which the task executor is blocked. Equals to CPU time if blocking IO isn't used.

### Log events

* Counter `elfo_emitted_events_total{level}`

    The number of emitted events per level (`Error`, `Warn`, `Info`, `Debug`, `Trace`).

* Counter `elfo_limited_events_total{level}`

    The number of events that haven't been emitted because the limit was reached.

* Counter `elfo_lost_events_total{level}`

    The number of events that hasn't been emitted because the event storage is full.

### Dump events

* Counter `elfo_emitted_dumps_total`

    The number of emitted dumps.

* Counter `elfo_limited_dumps_total`

    The number of dumps that haven't been emitted because the limit was reached.

* Counter `elfo_lost_dumps_total`

    The number of dumps that hasn't been emitted because the dump storage is full.

### Other metrics

TODO: specific to elfo-logger, elfo-dumper, elfo_telemeter

## Derived metrics

### Statuses

TODO

### Incoming/outgoing rate

TODO

```
rate(elfo_message_handling_time_seconds_count{actor_group="${actor_group:raw}",actor_key=""}[$__rate_interval])
```

### Waiting time

TODO

```
rate(elfo_message_waiting_time_seconds{actor_group="${actor_group:raw}",actor_key="",quantile=~"0.75|0.9|0.95"}[$__rate_interval])
```

### Utilization

TODO


```
rate(elfo_message_handling_time_seconds_sum{actor_group="${actor_group:raw}",actor_key=""}[$__rate_interval])
```

### Executor utilization (≈ CPU usage)

TODO

The time for which the task executor is blocked. Equals to CPU time if blocking IO isn't used.

```
rate(elfo_busy_time_seconds_sum[$__rate_interval])
```

## Dashboards

TODO

## Implementation details

TODO
