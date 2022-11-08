# Rust errors

Error handling gets special treatment in the language because when reading and writing code it is convenient to understand the happy path and the error path separately. Since errors are exceptional, we usually want to read code assuming errors aren't occurring, so making error handling very low-overhead is useful. Furthermore, by supporting error handling in the language and standard library, different libraries can handle errors without requiring loads of boilerplate for translating errors at library boundaries.

Rust has two ways to represent errors: using the `Result` type and by panicking. The former is much more common and usually preferred.

The `Result` type is a regular enum with `Ok(T)` and `Err(T)` variants. The former signaling normal program behaviour, and the latter signaling an error. Both variants contain regular Rust values and the programmer can freely choose the types for both. There is essentially no special mechanism here. A function returns just a regular value which can indicate either success or an error; the callers of a function must check which. Propagating an error simply means returning an error when an error is found. Rust does provide some special constructs to make this easy.

Panicking is a special mechanism for 'immediately' stopping progress in a controlled manner. It is triggered by macros like `panic!` and functions like `unwrap`. It can also be triggered implicitly, e.g., when arithmetic overflows. Panicking is usually not handled by the programmer and terminates the current thread.

The facilities for errors in the language and standard library are incomplete. There are several crates that can help with error handling and you'll probably want to use one.

## [Result and Error](result-and-error.md)


Covers the `Result` type and using it for error handling, the `?` operator, the `Error` trait, the `Try` trait, and other parts of Rust's machinery for handling errors as values.

## [Panic](panic.md)


Covers panicking, the panic macros, and panicking functions.

## [Non-Rust errors](interop.md)


How to deal with errors that originate outside of Rust, primarily when interoperating with other languages.

## [Testing](testing.md)


How to test code that uses `Result` or panics.
