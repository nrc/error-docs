# Non-Rust errors

If you interoperate with other languages or interact with the operating system in certain ways (usually this means avoiding Rust's existing abstractions in the standard library or crates), then you will need to deal with errors which are not part of Rust's error system. These fall into two categories: error values and exceptions. In both cases, you must convert these errors into Rust errors, this chapter will explain how, as well as going in the other direction - handling Rust errors at the FFI boundary.

## C errors

C has no standardized way to handle errors. Different libraries will indicate errors in different ways, e.g., by returning an error number, null pointer, sentinel value, etc. They may also set a global variable to give more error context. At the FFI boundary you must convert these different kinds of errors into Rust errors.

Generally, for FFI you will have a -sys crate which has simple bindings for foreign functions in Rust. This crate will have the native error handling. Then you will have a crate which wraps these function bindings in a Rust library which provides a safe, idiomatic Rust API. It is in this crate that errors should be translated into Rust errors, usually `Result`s. Exactly how you do this depends on the level of abstraction your crate is targetting. You may have a direct representation of the C error in your Rust error, e.g., return `Result<T, ErrNo>` where `ErrNo` is an enum with a variant for each error number, or even a newtype or alias of the integer type. At the other extreme, you might use an error type like any other Rust crate. In between, you might have a typical Rust error type and embed a representation of the underlying C error inside the Rust error. Essentially, dealing errors are 'just data' in both C and Rust, so translating errors at the FFI boundary is similar to handling any other data at the boundary.


## OS errors

You might interact with the operating system via the standard library or a crate, at the level of an API or syscalls. In these cases you might need direct access to the operating system's errors. In the standard library, these are available from `io::Error`. 

You can use `std::io::Error::raw_os_error` and `std::io::Error::from_raw_os_error` to convert between a Rust error and an operating system's error number. Some operating system operations do not return an error, but instead return some indication of an error (such as a null pointer) and make more information about the error available in some other way. `std::io::Error::last_os_error` provides a way to access that information as a Rust error. You must be careful to use it immediately after producing the error; other standard library functions or accessing the operating system outside of the standard library might change or reset the error information.


## Exceptions and panics

Languages often provide mechanisms for error handling which are not simple return values. Exceptions in many languages and Rust panics are examples. These mechanisms need special handling at the FFI boundary. In general, anything which involves stack unwinding or non-local control flow (that is, where execution jumps from one function to another) must not cross the FFI boundary. That means that on the Rust side, panics must not unwind through a foreign function (i.e., panics must be *caught*); if they do, the process will abort. For other languages, exceptions and similar must not jump into or 'over' (on the stack) Rust code; this will cause undefined behaviour.

If Rust code might panic (and most Rust code can) and is called by foreign code, then you should catch any panics using the [`catch_unwind`](https://doc.rust-lang.org/stable/std/panic/fn.catch_unwind.html) function. You may also find `std::panic::resume_unwind` and `std::thread::panicking` useful. 

There is work in progress to permit unwinding from Rust into C code using the `c_unwind` (and other `_unwind`) APIs. However, at the time of writing this work is not fully implemented and is unstable. See [RFC 2945](https://github.com/rust-lang/rfcs/blob/master/text/2945-c-unwind-abi.md) for more details.
