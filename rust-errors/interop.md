# Non-Rust errors

If you interoperate with other languages or interact with the operating system directly (i.e., avoiding the standard library abstractions or similar), then you will need to deal with errors which are not part of Rust's error system. These fall into two categories: error values and exceptions. This chapter will explain how to convert foreign errors into Rust errors and how to deal with exceptions.

## C errors

C has no standardized way to handle errors. Different libraries will indicate errors in different ways, e.g., by returning an error number, null pointer, sentinel value, etc. They may also set a global variable to give more error context. At the FFI boundary you must convert these different kinds of errors into Rust errors.

A common and useful pattern is to have a `-sys` crate which has simple, direct bindings for foreign functions in Rust (and could be mechanically generated). This crate will have native error handling (e.g., returning a `usize` as a success/error code, or an `Option` which corresponds with a possibly-null pointer). Then you will have a crate which wraps these low-level bindings in a Rust library which provides a safe, idiomatic Rust API. It is in this crate that errors should be translated into Rust errors, usually `Result`s.

Exactly how you do this depends on the level of abstraction the idiomatic crate is targeting. You may have a direct representation of the C error in your Rust error, e.g., return `Result<T, ErrNo>` where `ErrNo` is a newtype wrapping the integer type or an alias of it. At the other extreme, you might use an error type like any other Rust crate. Or you might have a typical Rust error type and embed a representation of the underlying C error inside the Rust error. Essentially, errors are 'just data' in both C and Rust, so translating errors at the FFI boundary is similar to translating any other data at the boundary.

A concrete example of such a translation:

```rust,ignore
fn map_err_code(code: isize) -> Result<usize, MyError> {
    match code {
        c if c >= 0 => Ok(c as usize),
        -1 => Err(MyError::Foo),
        -2 => Err(MyError::Bar),
        -3 | -4 => Err(MyError::Baz),
        c => Err(MyError::UnknownForeign(c)),
    }
}
```

Depending on the context, you might want to implement this as a `From` impl rather than a stand-alone function.

In the opposite direction, you might need to convert Rust errors into C errors (or errors in other languages). This is usually done with a simple match to convert the Rust error into a Rust representation of whatever the foreign errors look like (e.g., an `Option` for a null pointer, or an integer of some kind for an error code).

## OS errors

You might interact with the operating system via the standard library or a crate, or using syscalls. In these cases you might need access to the operating system's errors. In the standard library, these are available from [`io::Error`](https://doc.rust-lang.org/stable/std/io/struct.Error.html).

You can use `std::io::Error::raw_os_error` and `std::io::Error::from_raw_os_error` to convert between a Rust error and an operating system's error number. Some operating system operations do not return an error, but instead return some indication of an error (such as a null pointer) and make more information about the error available in some other way. `std::io::Error::last_os_error` provides a way to access that information as a Rust error. You must be careful to use it immediately after producing the error; other standard library functions or accessing the operating system outside of the standard library might change or reset the error information.


## Exceptions and unwinding

Languages often provide mechanisms for error handling which are not simple return values. Exceptions in languages such as Java or C++, or `setjmp`/`longjmp` in C are examples. These mechanisms are another flavor of unwinding and need special consideration at the FFI boundary.

The [ABIs](https://doc.rust-lang.org/reference/items/functions.html#extern-function-qualifier) for an `extern` function declared in Rust come in two variants[^rust-abi]: with and without an `-unwind` suffix. If using the `-unwind` ABIs, then foreign unwinds may cross the FFI boundary into Rust code. If `panic = "abort"`, the process will terminate if a foreign unwind reaches the FFI boundary. Otherwise, unwinding will continue into the Rust code. If caught in Rust using [`catch_unwind`](https://doc.rust-lang.org/stable/std/panic/fn.catch_unwind.html), then either you will get an opaque error, or the process will abort (which one happens is unspecified).

If using a non-unwind ABI, then any foreign unwind reaching the FFI boundary will cause *undefined behavior*. Any Rust panic reaching the FFI boundary will abort the process (even if not using `panic = "abort"`). Therefore, you must ensure that unwinding from foreign code does not reach Rust code by catching the unwind in the foreign code. You must ensure Rust unwinding does not reach foreign code by catching unwinding, e.g., by using `catch_unwind` or [`std::thread::panicking`](https://doc.rust-lang.org/stable/std/thread/fn.panicking.html).

There is another kind of unwinding: [forced unwinding](https://rust-lang.github.io/rfcs/2945-c-unwind-abi.html#forced-unwinding), for example `pthread_cancel` on Linux. No matter the ABI, forced unwinding which crosses an FFI boundary is undefined behavior and thus must be prevented.

There are also some edge cases with unwinding and FFI which are undefined behavior and must be avoided:

- If a function is declared using a non-unwinding ABI, but is called from within a function with unwinding ABI, it is undefined behavior for unwinding to cross the FFI boundary.
- Calling a Rust function declared with an unwinding ABI, from foreign code without unwinding support (e.g., code compiled with `-fno-exceptions`) is undefined behavior if it unwinds.
- When considering whether unwinding will cause problems, you must consider all the stack frames that get unwound through/over, not just where the unwinding starts and ends.

[^rust-abi]: Except the `rust` ABI which always and implicitly supports unwinding.
