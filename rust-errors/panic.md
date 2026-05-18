# Panic

Panicking is a mechanism for crashing a thread in an orderly way. Unlike `Result`-based errors or exceptions in other languages, panics are not intended to be caught or handled (although they sometimes can be).

By default, panicking terminates the *current thread* by *unwinding* the stack, executing all destructors (`Drop::drop`) as it goes. This means that the program can be left in a consistent state and the rest of the program can carry on executing.

Panicking is a great feature:

- It is a disciplined and reliable way to clean up when something goes wrong.
- It is not a crash and is not exploitable.
- It can be isolated to a single thread.

But it has downsides:

- It is intrinsically implicit and non-local, thus hard to reason about.
- It can harm performance by introducing another path through execution that the optimizer must take into account.
- If you recover from panics (even on another thread), data can be left in an inconsistent state.
- It can be triggered in many places due to indexing out of bounds, integer overflow, etc.
- Double panicking causes the whole program to terminate, and this can be hard to predict or prevent.

Panicking is not a bad thing. However, programmers often underestimate the amount of work required to ensure correctness in the presence of panics. Using `Result` for error handling (and reserving panicking for 'impossible' situations) is usually a better idea.

It's important for programmers to understand that panicking is nearly always possible in Rust code. It's an important edge case which should be considered in design, implementation, and testing.


## Abort on panic

You can configure your program to abort on panic (by adding `panic = "abort"` to the appropriate profile in your [Cargo.toml](https://doc.rust-lang.org/book/ch09-01-unrecoverable-errors-with-panic.html#unwinding-the-stack-or-aborting-in-response-to-a-panic)). In this case, the whole program exits immediately when a panic occurs. Using abort-on-panic will make your program slightly more performant (because stack unwinding doesn't need to be considered, similar to using C++ without exceptions). However, it can make your program less robust (a single thread cannot crash without crashing the whole program), and it means destructors won't be run on panic.

Even in the default unwind-on-panic configuration, causing a panic while the current thread is already panicking will cause the program to abort. You must therefore be very careful that destructors cannot panic under any circumstance. You can check if the current thread is panicking by using the [`std::thread::panicking`](https://doc.rust-lang.org/stable/std/thread/fn.panicking.html) function.


## Triggering panics

Panics can be triggered in all sorts of ways. Most obviously by using the [`panic!`](https://doc.rust-lang.org/stable/std/macro.panic.html) macro, which unconditionally starts a panic when it is executed. The `todo!`, `unimplemented!`, and `unreachable!` macros all do the same thing, but with different implications for human readers (the [`unreachable_unchecked`](https://doc.rust-lang.org/stable/std/hint/fn.unreachable_unchecked.html) function does not panic; it is an indication to the compiler that its location is never reachable, and executing it is undefined behaviour). [`panic_any`](https://doc.rust-lang.org/stable/std/panic/fn.panic_any.html) is a function similar to the `panic!` macro which lets you panic with an arbitrary object, rather than a string message.

The `assert_...` and `debug_assert_...` macros panic if their conditions are not met. E.g., [`debug_assert_eq`](https://doc.rust-lang.org/stable/std/macro.debug_assert_eq.html) panics (only in debug builds) if its arguments are not equal.

The functions `unwrap` and `expect` on `Result` or `Option` panic if the receiver is `Err` or `None` (respectively). They can be thought of as a bridge between the worlds of `Result` and panics. Because it's so common, it's worth calling out the `lock().unwrap()` idiom for handling mutex poisoning by propagating panics across threads.

Indexing (e.g., `some_slice[42]`) out of bounds (or for some key which does not exist) causes a panic. In debug builds, arithmetic overflow or underflow causes a panic. There are many functions in the standard library which can panic if something goes wrong:

- Methods on `RefCell`, `Cell`, etc. such as `RefCell::borrow` on violations of their borrowing invariants.
- `push` and similar methods on collections when their capacity overflows.
- `Iterator::step_by(0)`.
- Any function which can allocate if the allocator panics on failure (which is usually only in `no_std` builds).

In general, since panicking is not part of the type system or annotated on function signatures, you must assume that any function call might cause a panic, unless you can prove otherwise.


## Is panicking safe?

Given that panicking is similar to a crash and crashes are often exploitable, people often ask if panicking is safe. There are many levels to this question! Importantly, panicking cannot cause memory unsafety, so panicking is safe in the sense of Rust's `unsafe` keyword. Similarly, panicking is never exploitable. Since panicking runs destructors, panicking is also fairly safe in a colloquial sense since it can't leave your program in an inconsistent state (although you still have to take care to avoid bugs here, as mentioned above). However, if panicking causes your whole program to crash (due to a double panic, propagating panics to other threads, or abort-on-panic), then this is a pretty poor user experience.


## Programming with panics[^blog]

You should not use panicking for general error handling. Specifically, you should only use panics for things which (in theory) can't happen. That is, every time a panic does happen, it is a bug in the program (or a cosmic ray or something). It's important to differentiate something which is truly impossible (e.g., a function returning an enum variant which it never creates) from something which is very unlikely (e.g., a string read from disk not being valid UTF8). You'll need to take care to avoid `unwrap`ing unless you're really sure it's impossible to be `Err`/`None`. Likewise, if you're unsure that an index or key is in-bounds, use `get` rather than indexing. But when something *is* impossible, don't avoid panics.

User-supplied data (either from an end-user or client code) should never cause panics. Any error in the data itself or the processing of it should create an error which is returned to the user, rather than panicking.

In the case of library crates, the crate should be agnostic to how client code treats panics. Data should be resilient to incomplete clean-up, interop code should handle panics and exceptions properly, user panics should not be able to poison library mutexes (or vice versa), etc. The crate should also document its requirements with respect to panics. 

One thing you might have to consider is the state of other threads. Panicking terminates a thread so you don't need to worry about data which is only referenced from the panicked thread. But data which is referenced from multiple threads could be corrupted. You can use destructors to ensure that shared data is left in a consistent state since these will be run on panic. However, you cannot rely on destructors being run in all circumstances (e.g., consider an object referred to by an `Arc` from another thread).

An example of a feature for ensuring consistency in the face of a panicking thread is [mutex lock poisoning](https://doc.rust-lang.org/stable/std/sync/struct.Mutex.html#poisoning). When you lock a mutex, you get a `Result` which will be an `Err` if another thread panicked while the lock was held. If you `unwrap` this result, then you essentially propagate the panic to the current thread (which is a fine approach). Alternatively, you could try to recover by fixing any inconsistent state and un-poisoning the mutex[^non-poisoning].

General advice:

- Try to minimise potential panics in your code.
- When deciding if a panic is appropriate, try to keep reasoning about invariants as local as possible.
- Document whether a function can panic and in what circumstances.
- Remember panicking as an edge case when writing code, ensure that panicking code paths are appropriately tested.

Opinionated advice:

- Never catch, handle, or recover from panics. Any panic should terminate the whole program (not just a thread).
  - Always `unwrap` `Mutex::lock`.
  - FFI is an exception, if necessary.
  - Use `panic = "abort"`.
- Don't rely on destructors running or on general clean-up code (it is not guaranteed to run).

"Any panic should terminate the whole program" deserves some explanation: the thinking here is that panics often lead to tricky inconsistency of state and that recovering from that (in non-trivial programs) is difficult, perhaps impossible to do in a bug-free way. Better for the program to terminate and start again from scratch (because non-ephemeral data is all persisted and restarting and carrying on with tasks is supported, right?). This requires that there is some kind of supervisor process which can restart the terminated program (common for server-side software) or that rerunning the program is trivial for the user (true for some CLI software). GUI software and other user-facing programs can be a poor fit for this strategy. It also requires that restarting is not going to cause the exact same panic which terminated the program. That can lead to an infinite loop of crashes and restarts. If possible, you'll want to log the restart with as much detail as possible about the reason to help with debugging.

If you can't restart and need to be more resilient, then I have more opinions:

- Try harder! But if you really can't...
- Prefer to let the thread (or async task) die rather than catching panics close to the sources. The more coarse-grained the recovery, the easier things will be.
- Prefer to make data intrinsically resilient to inconsistency rather than relying on mutex poisoning.
- Check for panics happening in destructors if there is any chance at all of panicking.
- Even with all of the above, still try to minimise reliance on panics because double-panicking, inconsistent data, or other rare situations can crop up surprisingly often.


### Eliminating panics

You might want to ensure that your program (or parts of your program) cannot panic. However, this is generally difficult. If you don't want to have unwinding, then you can abort on panic. However, wanting your program to be panic-free is essentially wanting your program to be bug free (at least for several classes of bugs), which is a big ask! It is usually better for a bug to cause a panic rather than to cause incorrect execution (or undefined behaviour). In some circumstances, you might be able to check every arithmetic and indexing operation, handle every error, and never `unwrap` an `Option`, but for most programs, this is more of a burden than it is worthwhile. There are some crates ([dont_panic](https://github.com/Kixunil/dont_panic) and [no-panic](https://github.com/dtolnay/no-panic)) which can help with this, but they rely on causing a linking error, so only work in some circumstances, can depend on specific optimizations, and can't do anything smart like prove that a panic trigger is unreachable.

Pretending you are writing panic-free code by avoiding explicit panics is just wishful thinking!


[^blog]: I wrote a blog post about panics: [To panic or not to panic](https://www.ncameron.org/blog/to-panic-or-not-to-panic/). There's a lot of overlap with this section, but the blog post has a little more detail.

[^non-poisoning]: There is an experimental non-poisoning mutex available with nightly Rust: [nonpoison::Mutex](https://doc.rust-lang.org/nightly/std/sync/nonpoison/struct.Mutex.html). This might be appropriate for your project, but given that it is experimental and doesn't propagate panicking to other threads, I would only use it if you're using `panic = "abort"`, or it is impossible for the guarded value to be in an inconsistent state.


## Catching panics

It is possible to catch a panic, at least an unwinding one, using [`catch_unwind`](https://doc.rust-lang.org/stable/std/panic/fn.catch_unwind.html). That turns a panic into a `Result::Err` so you can handle it like any other `Result`. However, as stated in the docs, you probably shouldn't do this (unless you're interoperating with code in another language, see [below](#interop-and-catching-panics)). It's generally a bad idea because:

- It only works when unwinding, if you're using `panic = "abort"`, the program will still abort.
- Using `Result` for error handling is best practice in Rust. Using panics for error handling will make it harder to integrate with the ecosystem and be surprising to people reading your code.
- Because there is an expectation that unwinding won't be caught and a whole thread will be unwound and dropped, then if you catch panics, you might be violating the assumptions of other programmers (in particular, in `unsafe` code). See discussion of `UnwindSafe` below.
- Even if you catch a panic, the panic hook is still triggered, so recovery from a panic is never 'clean'.

There is an [`UnwindSafe`](https://doc.rust-lang.org/stable/std/panic/trait.UnwindSafe.html) bound on the code executed by `catch_unwind`. This is meant to ensure that any invariants in that code or on data structures used in it will behave consistently if an unwinding panic is caught, i.e., the code is 'unwind-safe'. Note that all Rust code *should* be memory-safe in the presence of unwinding; `UnwindSafe` indicates that types behave correctly in general, as well as just with respect to memory safety.

If you are writing unsafe code, then you must ensure that your code is [exception safe](https://doc.rust-lang.org/nomicon/exception-safety.html). This means that any safety invariants your code relies on are re-established at any point where the program might panic. To be `UnwindSafe` that should apply to your correctness invariants as well as your memory safety invariants.

Unfortunately, the concept of `UnwindSafe` has not gained a wide following in the Rust community, so it is probably unwise to rely on it too much.


### Interop and catching panics

Panics must not cross the FFI boundary. That means that you must catch Rust panics using [`catch_unwind`](https://doc.rust-lang.org/stable/std/panic/fn.catch_unwind.html). See the [interop chapter](interop.md) for details.


## Backtraces

When a panic happens, Rust can print a backtrace of the stack frames which led to the panic (i.e., all the functions which were being called when the panic happened). To show the backtrace, set the `RUST_BACKTRACE` environment variable to `1` (or `full` if you want a verbose backtrace) when you run your program. You can prevent backtraces being printed (or change their behaviour) by setting your own [panic hook](https://doc.rust-lang.org/nightly/std/panic/fn.set_hook.html).

You can capture and manipulate backtraces programmatically using [`std::backtrace`](https://doc.rust-lang.org/nightly/std/backtrace/index.html).


## Panic hooks and handlers

When a panic occurs, a *panic hook* function is called (before unwinding, and thus running destructors, or aborting). By default, this prints a message and possibly a backtrace to `stderr`, but it can be customised. The panic hook is called whether the panic will unwind or abort.

In a `no_std` crate, you'll need to set your own panic handler. Use the [`#[panic_handler]`](https://doc.rust-lang.org/nomicon/panic-handler.html) attribute and see the linked docs for more info.

For more details on how panicking is implemented, see this [blog post](https://www.ralfj.de/blog/2019/11/25/how-to-panic-in-rust.html); for more on if and when you should panic (particularly using `unwrap` see [this one](https://blog.burntsushi.net/unwrap/).
