# Rust errors

Error handling is an important part of programming. It is essential that your program behaves well, even when something goes wrong. Error handling gets special treatment in the language because when reading and writing code it is convenient to understand the happy path and the error path separately. Since errors are exceptional, we usually want to read code assuming errors aren't occurring, so making error handling implicit (or very low-overhead) is useful. Furthermore, by supporting error handling in the language and standard library, different libraries can handle errors without requiring loads of boilerplate for translating errors at library boundaries.

Rust has two ways to represent errors: the `Result` type and by panicking. The `Result` type is a regular enum with `Ok` and `Err` variants. The former signalling normal program behaviour, and the latter signalling an error. Both variants contain regular Rust values and the programmer can freely choose the types of both. Panicking is a special mechanism for immediately stopping progress. It is triggered by macros like `panic!` and functions like `unwrap`. It can also be triggered implicitly, e.g., when arithmetic overflows. Panicking is usually not handled by the programmer and terminates the current thread.

TODO I wonder if these chapters are too abstract? Should it have a more practical guide to throwing and recovering from errors?

## Result and Error

[result-and-error.md](result-and-error.md)

Covers the `Result` type and using it for error handling, the `?` operator, the `Error` trait, the `Try` trait, and other parts of Rust's machinery for handling errors as values.

## Panic

[panic.md](panic.md)

Covers panicking, the panic macros, and panicking functions.

## Non-Rust errors

## Testing
