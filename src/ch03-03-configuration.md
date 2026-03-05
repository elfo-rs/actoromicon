# Configuration

## System messages

- `ValidateConfig`. This message is designed to validate config against actor's state. Any other validation should happen on the deserialization stage. Consider using [validator](https://lib.rs/crates/validator) for complex validation at deserialization time.

TODO
