# Result and Error

Using a `Result` is the primary way of handling errors in Rust. A `Result` is an enum in the standard library, it is not part of the language or special in any way. It has two variants: `Ok` and `Err`, which indicate correct execution and incorrect execution, respectively. Both `Result` and its two variants are in the prelude, so you don't need to explicitly import them and you don't have to write `Result::Ok` as you would for most enums.

Both variants take a single argument which can be any type, you'll usually see `Result<T, E>` where `T` is the type of value returned in `Ok` and `E` is the error type returned in `Err`. It's also fairly common for modules to use a single error type for the whole module and to define an alias like `pub type Result<T> = std::result::Result<T, MyError>;`. In that case you'll see `Result<T>` as the type where `MyError` is an implicit error type.

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

There are a lot more ergonomic ways to handle `Result`s, however. There are a whole load of combinator methods on `Result`. See the [docs](https://doc.rust-lang.org/stable/std/result/enum.Result.html#implementations) for details. There are way too many to cover here, but just as an example, `map` takes a function and applies it to the payload of the result if the result is `Ok` and does nothing if it is `Err`, e.g.,

```
fn foo(r: Result<i32, MyError>) -> Result<String, MyError> {
    r.map(|i| i.to_string())
}
```

There is also the `?` operator; this is extremely common in Rust code and often the most ergonomic way to work with `Result`s. Applying `?` to a result will either unwrap the payload, if the result is `Ok`, or immediately return the error if it is `Err`. E.g.,

```
fn foo(r: Result<i32, MyError>) -> Result<String, MyError> {
    let i = r?;
    Ok(i.to_string())
}
```

The above code does the same as the previous example. In this case, the `map` code is more idiomatic, but if we had a lot of work to do with `i`, then using `?` would be better. Note that the type of `i` is `i32`.

The `?` operator can be used with other types too, notably `Option`. You can also use it with your own types, see the discussion of the `Try` trait, below.

One final feature of `?` does not have to just return the error type directly. It will call `Into::into` on the error type. So if you have two error types: `Error1` and `Error2` and `Error1` implements `Into<Error2>`, then the following will work (note the different error types in `Result`):

```
fn foo(r: Result<i32, Error1>) -> Result<String, Error2> {
    let i = r?;
    // ...
}
```

This implicit conversion is relied on with several patterns of error handling we'll see in later chapters.

## `Option`

`Option` is similar to `Result` in that it is a very common, two-variant enum type with a payload type (`Some`, c.f., `Ok`). The difference is that `Option::None` (c.f., `Result::Err`) does not carry a payload. It is, therefore, structurally equivalent (theoretically, not necessarily at runtime) to `Result<T, ()>` and there are many methods for converting between `Option` and `Result`. `?` works with `Option` just like `Result`.

The intended semantics of the types, however, are different. `Result` represents a value which has either been correctly computer or where an error occurred in its computation. `Option` represents a value which may or may not be present. Generally, `Option` should not be used for error handling. You may use or come across types like `Result<Option<i32>, MyError>`, this type might be returned where the computation is fallible, and if succeeds it will return either an `i32` or nothing (but importantly, returning nothing is not an error).

## The `Error` trait

TODO

  backtraces (`Backtrace` trait), source
  dyn vs concrete errors, downcasting (lifetimes in error types)

## The `Try` trait

TODO and non-Result error handling

## Deprecated stuff

There is a deprecated macro, `try`, which does basically the same thing as `?`. It only works with `Result`, no other types, and in editions since 2018, you have to use `r#try` syntax to name it. It is deprecated and there is no reason to use it rather than `?`, you might come across it in old code though.

There are some deprecated methods on the `Error` trait. Again, there is no reason to use these, but you might see them in older code. `Error::description` has been replaced by using a `Display` impl. `Error::cause` has been replaced by `Error:source`, which has an additional `'static` bound on the return type which, allows it to be downcast to a concrete error type.
