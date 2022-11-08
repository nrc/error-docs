# Panic

Panicking is a mechanism for crashing a thread in an orderly way. Unlike `Result`-based errors or exceptions in other languages, panics are not intended to be caught or handled.

By default, panicking terminates the *current thread* by *unwinding* the stack, executing all destructors as it goes. This means that the program can be left in a consistent state and the rest of the program can carry on executing.

You can also configure your program (by adding `panic = 'abort'` to the appropriate profile in your [Cargo.toml](https://doc.rust-lang.org/book/ch09-01-unrecoverable-errors-with-panic.html#unwinding-the-stack-or-aborting-in-response-to-a-panic)) to abort on panic. In this case, the whole program exits immediately when a panic occurs. Using abort-on-panic will make your program slightly more performant (because stack unwinding doesn't need to be considered, similar to using C++ without exceptions). However, it can make your program less robust (a single thread cannot crash without crashing the whole program) and it means destructors won't be run on panic.

Even in the default unwind-on-panic configuration, causing a panic while the thread is already panicking will cause the whole program to abort. You must therefore be very careful that destructors cannot panic under any circumstance. You can check if the current thread is panicking by using the `std::thread::panicking` function.

When a panic occurs, a *panic hook* function is called. By default, this prints a message and possibly a backtrace to stderr, but it can be customised. See the docs for [`std::panic`](https://doc.rust-lang.org/nightly/std/panic/index.html) for more information. The panic hook is called whether the panic will unwind or abort.

In a no-std crate, you'll need to set your own panic handler. Use the [`#[panic_handler]`](https://doc.rust-lang.org/nomicon/panic-handler.html) attribute and see the docs for [`core::panicking`](https://doc.rust-lang.org/nightly/core/panicking/index.html) for more info.

For more details on how panicking is implemented, see this [blog post](https://www.ralfj.de/blog/2019/11/25/how-to-panic-in-rust.html).

## Triggering panics

Panics can be triggered in all sorts of ways. The most obvious way is using the `panic!` macro, that unconditionally starts a panic when it is encountered. The `todo!`, `unimplemented!`, and `unreachable!` macros all do the same thing, but with different implications for human readers (the `unreachable_unchecked` macro does not panic; it is an indication to the compiler that its location is never reachable, and encountering it is undefined behaviour). `panic_any` is a function similar to the `panic!` macro which lets you panic with an arbitrary object, rather than a string message.

The assert and debug_assert macros panic if their condition is not met. E.g., `debug_assert_eq` panics (only in debug builds) if it's arguments are not equal.

The functions `unwrap` and `expect` on `Result` or `Option` panic if the receiver is `Err` or `None` (respectively). They can be thought of as a bridge between the worlds of errors using `Result` and errors using `Result`.

There are many places where panicking is implicit. In debug builds, arithmetic overflow/underflow causes a panic. Indexing (e.g., `some_slice[42]`) out of bounds (or for a key which does not exist) causes a panic. In general, since panicking is not part of the type system or annotated on function signatures, you must assume that any function call might cause a panic, unless you can prove otherwise.

## Programming with panics

You should not use panicking for general error handling. If you follow this advice, then most programmers won't have to worry too much about panics, in general, it just works. Specifically, you should only use panics for things which (in theory) can't happen. That is, every time a panic does happen, it is a bug in the program. You'll need to take some care to avoid `unwrap`ing things unless you're really sure it's impossible to be `Err`/`None`. Likewise, if you're unsure that an index or key is in-bounds, use `get` rather than indexing.

One thing you might have to consider is the state of other threads. Panicking terminates a thread so you don't need to worry about data which is only referenced from the panicked thread. But data which is referenced from multiple threads could be corrupted (note that this data must be `Sync`). You can use destructors to ensure that shared data is left in a consistent state since these will be run on panic. However, you cannot rely on destructors being run in all circumstances (e.g., consider an object referred to by an `Arc` from another thread).

An example of a feature for ensuring consistency in the face of a panicking thread is [mutex lock poisoning](https://doc.rust-lang.org/nightly/std/sync/struct.Mutex.html#poisoning). When you lock a mutex, you get a `Result` which will be an `Err` if another thread panicked while the lock was held. If you `unwrap` this result, then you essentially propagate the panic to the current thread (which is a fine approach). Alternatively, you could try to recover by fixing any inconsistent state.

If you are writing unsafe code, then you must ensure that your code is [exception safe](https://doc.rust-lang.org/nomicon/exception-safety.html). This means that any safety invariants your code relies on are re-established at any point where the program might panic. The [`UnwindSafe`](https://doc.rust-lang.org/nightly/std/panic/trait.UnwindSafe.html) trait is used to mark types as being unwind-safe, that is the type can be used after a panic has occurred and the unwinding has been caught. It is used with `catch_unwind`, which is used to catch panics at FFI boundaries (see below). However, the concept has not gained a wide following in the Rust community, so it is probably unwise to rely on it.

You might want to ensure that your program (or parts of your program) cannot panic. However, this is generally not a good idea. If you don't want to have unwinding, then you can abort on panic. However, wanting your program to be panic free is essentially wanting your program to be bug free (at least for several classes of bugs), which is a big ask! It is usually better for a bug to cause a panic rather than to cause incorrect execution. In some circumstances, you might be able to check every arithmetic and indexing operation, handle every error, and never `unwrap` an `Option`, but for most programs, this is more of a burden than it is worthwhile. There are some crates ([dont-panic](https://github.com/Kixunil/dont_panic) and [no_panic](https://github.com/dtolnay/no-panic)) which can help with this, but they rely on causing a linking error, so only work in some circumstances, can depend on specific optimisations, and can't do anything smart like prove that a panic trigger is unreachable.

### Is panicking safe?

Given that panicking feels like a crash and crashes are often exploitable, it is often asked if panicking is safe. There are many levels to this question! Importantly, panicking cannot cause memory unsafety, so panicking is safe in the sense of Rust's `unsafe` keyword. Similarly, panicking is never exploitable. Since panicking runs destructors, panicking is also fairly safe in a colloquial sense in the sense that it can't leave your program in an inconsistent state (although you still have to take care to avoid bugs here, as mentioned above). However, if panicking causes your whole program to crash (due to a double panic, propagating panics to other threads, or abort-on-panic), then this can be a poor user experience.

### Interop and catching panics

Panics must not cross the FFI boundary. That means that you must catch Rust panics using [`catch_unwind`](https://doc.rust-lang.org/stable/std/panic/fn.catch_unwind.html). See the [interop chapter](interop.md) for details.

## Backtraces

When a panic happens, Rust can print a backtrace of the stack frames which led to the panic (i.e., all the functions which were being called when the panic happened). To show the backtrace, set the `RUST_BACKTRACE` environment variable to `1` (or `full` if you want a verbose backtrace) when you run your program. You can prevent backtraces being printed (or change their behaviour) by setting your own [panic hook](https://doc.rust-lang.org/nightly/std/panic/fn.set_hook.html).

You can capture and manipulate backtraces programmatically using [`std::backtrace`](https://doc.rust-lang.org/nightly/std/backtrace/index.html).
