# Result and Error

Using a `Result` is the primary way of handling errors in Rust. A `Result` is a generic enum in the standard library, it has two variants: `Ok` and `Err`, which indicate correct execution and incorrect execution, respectively. Both `Result` and its two variants are in the prelude, so you don't need to explicitly import them and you don't have to write `Result::Ok` as you would for most enums.

Both variants take a single argument which can be any type. You'll usually see `Result<T, E>` where `T` is the type of value returned in `Ok` and `E` is the error type returned in `Err`. It's fairly common for modules to use a single error type for the whole module and to define an alias like `pub type Result<T> = std::result::Result<T, MyError>;`. In that case, you'll see `Result<T>` as the type, where `MyError` is an implicit error type.

Creating a `Result` value is as simple as using the variant as a constructor, just like other enums. E.g, for a `Result<i32, i32>` (a `Result` with an `i32` payload and an `i32` error code), you would use `Ok(42)` and `Err(2)` to create `Result` values.

When receiving a `Result` object, you can address the ok and error cases by using a `match` expression, e.g.,

```
fn foo(r: Result<String, MyError>) {
    match r {
        Ok(s) => println!("foo got a string: {s}"),
        Err(e) => println!("an error occurred: {e}"),
    }
}
```

There are more ergonomic ways to handle `Result`s too. There are a whole load of combinator methods on `Result`. See the [docs](https://doc.rust-lang.org/stable/std/result/enum.Result.html#implementations) for details. There are way too many to cover here, but as an example, `map` takes a function and applies it to the payload of the result if the result is `Ok` and does nothing if it is `Err`, e.g.,

```
fn foo(r: Result<i32, MyError>) -> Result<String, MyError> {
    r.map(|i| i.to_string())
}
```

### The `?` operator

Applying `?` to a result will either unwrap the payload, if the result is an `Ok`, or immediately return the error if it is an `Err`. E.g.,

```
fn foo(r: Result<i32, MyError>) -> Result<String, MyError> {
    let i = r?; // Unwraps Ok, returns an Err
    Ok(i.to_string())
}
```

The above code does the same as the previous example. In this case, the `map` code is more idiomatic, but if we had a lot of work to do with `i`, then using `?` would be better. Note that the type of `i` is `i32` in both examples.

The `?` operator can be used with other types too, notably `Option`. You can also use it with your own types, see the discussion of the `Try` trait, below.

`?` does not have to just return the error type directly. It can convert the error type into another by calling `From::from`. So if you have two error types: `Error1` and `Error2` and `Error2` implements `From<Error1>`, then the following will work (note the different error types in the signature):

```
fn foo(r: Result<i32, Error1>) -> Result<String, Error2> {
    let i = r?;
    // ...
}
```

This implicit conversion is relied on for several patterns of error handling we'll see in later chapters.

### Try blocks

Try blocks are an unstable language feature (`#![feature(try_blocks)]`). The syntax is `try { ... }` where the block contains code like any other block. The difference compared to a regular block, is that a try block introduces a scope for the question mark operator. If you use `?` inside a try block, it will propagate the error to the result of the block immediately, rather than returning the error from the current function.

For example,

```rust
let result: Result<i32, ParseIntError> = try {
    "1".parse::<i32>()?
        + "foo".parse::<i32>()?
        + "3".parse::<i32>()?
};
```

At runtime, execution will stop after the second `parse` call and the value of `result` will be the error from that call. Execution will continue after the `let` statement, without returning.

## `Option`

`Option` is similar to `Result` in that it is a very common, two-variant enum type with a payload type `Some` (compared with `Result::Ok`). The difference is that `Option::None` (compared with `Result::Err`) does not carry a payload. It is, therefore, analogous to `Result<T, ()>` and there are many methods for converting between `Option` and `Result`. `?` works with `Option` just like `Result`.

The intended meaning of the types, however, is different. `Result` represents a value which has either been correctly computed or where an error occurred in its computation. `Option` represents a value which may or may not be present. Generally, `Option` should not be used for error handling. You may use or come across types like `Result<Option<i32>, MyError>`, this type might be returned where the computation is fallible, and if succeeds it will return either an `i32` or nothing (but importantly, returning nothing is not an error).

## The `Error` trait

The error trait, [`std::error::Error`](https://doc.rust-lang.org/nightly/std/error/trait.Error.html), is a trait for error types. There is no hard requirement, you can use a type in a `Result` and with `?` without implementing `Error`. It has some useful functionality, and it means you can use dynamic error handling using `dyn Error` types. Generally you *should* implement `Error` for your error types (there are no required methods, so doing so is easy); most error libraries will do this for you.

### Provided functionality

`Display` is a super-trait of `Error` which means that you can always convert an error into a user-facing string. Most error libraries let you derive the `Display` impl using an attribute and a custom format string.

The `Error` trait has an experimental mechanism for attaching and retrieving arbitrary data to/from errors. This mechanism is type-driven: you provide and request data based on its type. You use `request` methods (e.g., [`request_ref`](https://doc.rust-lang.org/nightly/std/error/trait.Error.html#method.request_ref)) to request data and the [`provide`](https://doc.rust-lang.org/nightly/std/error/trait.Error.html#method.provide) method to provide data, either by value or by reference. You can use this mechanism to attach extra context to your errors in a uniform way.

A common usage of this mechanism is for backtraces. A backtrace is a record of the call stack when an error occurs. It has type [`Backtrace`](https://doc.rust-lang.org/nightly/std/backtrace/struct.Backtrace.html) which has various methods for iterating over the backtrace and capturing the backtrace when an error occurs. To get a backtrace, use the `Error::request_ref` method, e.g., `if let Some(trace) = err.request_ref::<Backtrace>() { ... }`.

For an error to support backtraces, it must capture a backtrace when the error occurs and store it as a field. The error must then override the `Error::provide` method to provide the backtrace if it is requested. For more details on this mechanism see the [docs](https://doc.rust-lang.org/nightly/std/any/index.html#provider-and-demand) of the `Provider` trait which is used behind the scenes to implement it. (Note that there used to be a specific method for getting a backtrace from an error and this has been replaced by the generic mechanism described here).

[`Error::source`](https://doc.rust-lang.org/nightly/std/error/trait.Error.html#method.source) is a way to access a lower-level cause of an error. For example, if your error type is an enum `MyError` with a variant `Io(std::io::Error)`, then you could implement `source` to return the nested `io::Error`. With deep nesting, you can imagine a chain of these source errors, and `Error` provides a [`sources`](https://doc.rust-lang.org/nightly/std/error/trait.Error.html#method.sources) method[^1] to get an iterator over this chain of source errors.

[^1]: This and some other methods are implemented on `dyn Error`, rather than in the trait itself. That makes these methods usable on trait objects (which wouldn't be possible otherwise due to generics, etc.), but means they are *only* usable on trait objects. That reflects the expected use of these methods with the dynamic style of error handling (see the following section).

### Dynamic error handling

When using `Result` you can specify the concrete error type, or you can use a trait object, e.g., `Result<T, Box<dyn Error>>`. We'll talk about the pros and cons of these approaches in later chapters on designing error handling, for now we'll just explain how it works.

To make this work, you implement `Error` for your concrete error types, and ensure they don't have any borrowed data (i.e., they have a `'static` bound). For ease of use, you'll want to provide a constructor which returns the abstract type (e.g., `Box<dyn Error>`) rather than the concrete type. Creating and propagating errors works the same way as using concrete error types.

Handling errors might be possible using the abstract type only (using the `Display` impl, `source` and `sources` methods, and any other context), or you can downcast the error trait object to a concrete type (using one of the `downcast` methods). Usually, there are many possibilities for the concrete type of the error, you can either try downcasting to each possible type (the methods return `Option` or `Result` which facilitates this) or use the `is` method to test for the concrete type. This technique is an alternative to the common `match` destructuring of concrete errors.

### Evolution of the `Error` trait

Error handling in general, and the `Error` trait in particular, have been evolving for a long time and are still in flux. Much of what is described above is nightly only and many unstable features have changed, and several stable ones deprecated. If you're targetting stable Rust, it is best to mostly avoid using the `Error` trait and instead use an ecosystem alternative like [Anyhow](https://github.com/dtolnay/anyhow). You should still implement `Error` though, since the trait itself is stable and this facilitates users of your code to choose the dynamic path if they want. You should also be conscious when reading docs/StackOverflow/blog posts that things may have changed.

## The `Try` trait

The `?` operator and try blocks, and their semantics are not hard-wired to `Result` and `Option`. They are tied to the [`Try`](https://doc.rust-lang.org/nightly/std/ops/trait.Try.html) trait, which means they can work with any type, including a custom alternative to `Result`. We don't recommend using your own custom `Result`, we have in fact never found a good use case (even when implementing an error handling library or in other advanced cases) and it will make your code much less compatible with the ecosystem. Probably, the reason you might want to implement the `Try` trait is for non-error related types which you want to support ergonomic short-circuiting behaviour using `?` (e.g., `Poll`; although because of the implied semantics of error handling around `?`, this might also be a bad idea). Anyway, it's a kinda complex trait and we're not going to dive into it here, see the docs if you're interested.

## Deprecated stuff

There is a deprecated macro, `try`, which does basically the same thing as `?`. It only works with `Result` and in editions since 2018, you have to use `r#try` syntax to name it. It is deprecated and there is no reason to use it rather than `?`; you might come across it in old code though.

There are some deprecated methods on the `Error` trait. Again, there is no reason to use these, but you might see them in older code. `Error::description` has been replaced by using a `Display` impl. `Error::cause` has been replaced by `Error:source`, which has an additional `'static` bound on the return type (which allows it to be downcast to a concrete error type).
