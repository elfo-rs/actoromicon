# Introduction

This book aims to describe a superior approach to build heavily asynchronous applications based on the actor model.
Major part of the book is about the `elfo` framework and is illustrated with best practices of its usage.

The second part of the book tells you about the best corporate practices of asynchronous applications' architecture.
Most of this knowledge can't be applied right away and you should develop your own solution suitable for your task.


## Goals
* Assist in building fault-tolerant systems.
* Be performant enough for low-latency systems.
* Be observable, provide enough metrics to detect problems.
* Provide built-in support of exposing log events, dumps, metrics, and trace events.
* Distributing actors should be as simple as possible.

## Non-goals
* Provide the most performant way to communicate between actors.
* Provide any HTTP server.

## Features
* Asynchronous actors with supervision and custom life cycle.
* Two-level routing system: between actor groups and inside them (sharding).
* Multiple protocols: actors (so-called gates) can handle messages from different protocols.
* Multiple patterns of communication: regular messages, request-response (*TODO: subscriptions*).
* Config updating and distribution on the fly.
* Appropriate for both low latency and high throughput tasks.
* Tracing: all messages have `trace_id` that spread across the system implicitly.
* Telemetry (via the `metrics` crate).
* Dumping: messages can be stored for further debugging.
* Seamless distribution across nodes *TODO*.
* Utils for simple testing.
* Utils for benchmarking *TODO*.
