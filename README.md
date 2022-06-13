# nrc's error docs

An introduction to error handling in Rust.

Very work in progress. You are probably better off looking at these alternate resources:

* [The Book](https://doc.rust-lang.org/nightly/book/ch09-00-error-handling.html)
* [Error handling WG error patterns book draft](https://github.com/rust-lang/project-error-handling/blob/master/error-design-patterns-book/src/SUMMARY.md)
* [Draft doc on Rust's error handling model](https://hackmd.io/VN6AtpySR4Or_CV8b8XjRg)
* [BurntSushi's blog post](https://blog.burntsushi.net/rust-error-handling/)

# Contents

* [Rust errors](rust-errors/README.md)
  - [Result and Error](rust-errors/result-and-error.md)
    - The `Try` trait
  - [Panic](rust-errors/panic.md)
  - Non-Rust errors
  - Testing
  - no-std environments
* [Error design](error-design/README.md)
  - [Thinking about errors](error-design/thinking-about-errors.md)
  - Error handling
  - Error type design
* Rust's ecosystem
  - Good crates
  - Historic crates
* The future
  - The `Error` trait
  - Try blocks, yeet, etc.
  - panic effect
  - panic handler rfc (2070)
