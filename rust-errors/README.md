# Rust errors

Most programming languages have some features and/or set of conventions and patterns for error handling. Error handling gets special treatment because when reading and writing code it is convenient to understand the happy path and the error path separately. Since errors are exceptional, we usually want to read code assuming errors aren't occurring, so making error handling implicit or low-overhead is desirable.

Rust has two mechanisms for errors: using the `Result` type and by panicking. The former is much more common and usually preferred.

`Result` is just a regular enum with `Ok(T)` and `Err(E)` variants. The former signalling normal program behaviour, and the latter signalling an error. Both variants contain regular Rust values and the programmer can freely choose the types for both. There is no magic here. A function returns a value which can indicate either success or an error; the callers of a function must check which. Propagating an error simply means returning an error when an error is found. Rust does provide some language features to make this easier, we'll cover those in the [next chapter](result-and-error.md).

Panicking is a special mechanism for 'immediately' stopping progress in a controlled manner. It is triggered by macros like `panic!` and functions like `unwrap`. It can also be triggered implicitly, e.g., when arithmetic overflows. Panicking is usually not handled by the programmer and terminates the thread.

As well as the facilities in the language and standard library, there are several crates which can help with error handling and you'll probably want to use one.

## Section contents

### [Result and Error](result-and-error.md)

Covers the `Result` type and using it for error handling, the `?` operator, the `Error` trait, the `Try` trait, and other parts of Rust's machinery for handling errors as values.

### [Panic](panic.md)

Covers panicking, the panic macros, and panicking functions.

### [Non-Rust errors](interop.md)

How to deal with errors which originate outside of Rust, primarily when interoperating with other languages.

### [Testing](testing.md)

How to test code which uses `Result` or panics.
