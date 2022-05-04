# Panic

Panicking
  panic vs abort
  unwinding
  panics in dtors, thread::panicking
  PanicLocation, PanicInfo types
panic!, assert (and debug_assert) macros, panic_any
  file, column, and line macros
  todo, unimplemented, unreachable macros (cf core::hint::unreachable_unchecked)
unwrap, expect
Implicit panics
catching a panic
unwind safety
  types
  https://doc.rust-lang.org/nomicon/exception-safety.html
panic hooks
FFI
Backtraces, std::backtrace, env vars, backtrace style
panic implementation, core::panicking, `#[panic_handler]`
  https://www.ralfj.de/blog/2019/11/25/how-to-panic-in-rust.html


Panic can happen anywhere, panic-free code is a myth
  [no_panic](https://github.com/dtolnay/no-panic)
Inconsistent states, mutex poisoning
Is it safe? (In terms of security)
