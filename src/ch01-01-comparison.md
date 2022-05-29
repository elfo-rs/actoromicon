# Comparison aka "Why not X"?

## Where X = CSP

Usage of [CSP] in Rust can be illustrated in the following example:
```rust
// "Processes"
async fn read_file(path: &str, tx: Sender<Chunk>) { .. }
async fn decode_chunks(rx: Receiver<Chunk>, tx: Sender<SomeEvent>) { .. }
async fn process_events(rx: Receiver<SomeEvent>) { .. }

// "Channels"
let (chunks_tx, chunks_rx) = channel(100);
let (events_tx, events_rx) = channel(100);

spawn(read_file(path, chunks_tx));
spawn(decode_chunks(chunks_rx, events_tx));
spawn(process_events(events_rx));
```

The CSP approach is a perfect solution that doesn't require expertise in any frameworks for tools or simple applications with well-defined technical specifications and a small number of communications between processes. If this is your case, just use CSP.

However, complex applications tend to get more and more complicated over time, and their development and maintenance quickly become harder than in the actor model.

### Pros of CSP
* Implementation of channels can be chosen by a developer, while actor frameworks determine a mailbox implementation.
* Processes can share the same channel using MPMC channels to implement work-stealing behavior. It's not an option for actors, where a mailbox is owned by exactly one actor.

### Cons of CSP
* Processes in CSP are anonymous, while actors have identities. It means it's hard to distinguish logs and metrics because processes don't have names. Thus, the observability of CSP is much worse than that of the actor model.
* Actors are more decoupled; they can be discovered using some sort of service locators and even changed on the fly, e.g., due to restart.
* Actors can be distributed across several machines because they don't have to send messages directly to a mailbox; they can have a network before it.
* To add more connections between processes, we need to use more channels in one case and combine messages into big enumerations with unrelated items in other cases.

## Where X = actix

TODO

## Where X = bastion

TODO

[CSP]: https://en.wikipedia.org/wiki/Communicating_sequential_processes
