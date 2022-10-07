# Non-Rust errors

If you interoperate with other languages or interact with the operating system in certain ways (usually this means avoiding Rust's existing abstractions in the standard library or crates), then you will need to deal with errors which are not part of Rust's error system. These fall into two categories: error values and exceptions. In both cases, you must convert these errors into Rust errors, this chapter will explain how, as well as going in the other direction - handling Rust errors at the FFI boundary.

## OS errors

io::Error::{raw_os_error, from_raw_os_error}
io::Error::last_os_error

## C errors

TODO check ffi docs
kinds of error: null, error number, sentinel value, TODO
    convert to Rust errors at error boundary, perhaps including underlying error as an inner error
Rust errors -> C errors: just data, basically

## Exceptions

C++
other unwinding, long jump
unwinding from Rust
Java/.net exceptions?

catch_unwind
resume_unwind
thread::panicking
