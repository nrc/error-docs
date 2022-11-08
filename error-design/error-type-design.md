# Error type design

You can use any type as an error in Rust. The `E` in `Result<T, E>` has no bounds and can be anything. At the highest level there are two choices: use a concrete error type or use a trait object (e.g., `Box<dyn Error>`). In the latter case, you'll still need concrete error types to use for values, you just won't expose them as part of your API. When designing the concrete error types, there are two common approaches: create a taxonomy of error types primarily using enums, or use a single struct which represents all the errors in a module or modules.

There are also, some less common choices for the concrete error types. You can use an integer type which can be useful around FFI where you have an error number from C code. You should probably convert the error number into a more idiomatic error type before passing it on, but it can be a useful technique as an intermediate form between C and Rust types, or for some very specialised cases.

You can also use a string error message as your error type, this is rarely suitable for production code but is useful when prototyping because converting code from one kind of error type to another is much easier than converting panicking code to using `Result`s (it also lets you use `?` which can clean up your code nicely). You'll probably want to include `type Result<T> = std::result::Result<T, &'static str>;` to make this technique more usable. Alternatively, you can use string messages as the concrete types underlying trait object error types. Anyhow provides the `anyhow!`/`bail!` macro to make this easy.

For the rest of this section, we'll describe the two styles for concrete error types, then using trait objects as error types. We'll finish with some discussion on how to choose which technique to use, and some general tips for writing better error types.

## Enum style

In this style, you define a taxonomy of errors with a high level of detail and specialization, in particular, different kinds of errors have different data. Concretely, this means primarily using enums with a variant for each kind of error and the data specific to that kind of error embedded in the variant. E.g.,

```rust
pub enum MyError {
    Network(IoError, Request),
    Malformed(MalformedRespError),
    MissingHeader(String),
    UnknownField(String),
}

pub struct MalformedRespError {
    // ...
}
```

(Note that I think this is a poorly designed error, I'll give better designs below).

The advantages of this approach are:

* there is maximum information available, this is especially useful for error recovery,
* it's easy to extend with new errors,
* it's a common approach and is conceptually simple, so it is easy to understand,
* by using more enums or fewer enums with more variants, you can adjust the amount of static vs dynamic typing,
* throwing, rethrowing (especially at API boundaries when making errors strictly more abstract), and catching errors is ergonomic,
* it works well with error libraries like thiserror.

There are two ways to consider the design: by designing based on how the errors will arise or how the errors will be handled. I think the former is easier and more future-proof (how can you predict all the ways an error might be handled?).

There is a lot of scope for different design decisions: how to categorise errors, how much detail to include, how to structure the errors, how to handle back/forward compatibility, etc. Concretely, how many enums should you use and how many variants? Should you nest errors? How should you name the enums and variants?

If designing based on how errors arise, you want one enum for each class of errors and one variant for each specific kind of error (obviously, there's a lot of space here for what constitutes a class or specific kind of error). More practically, a function can only return a single error type, so all errors which can be returned from a function must belong to a single enum. Since errors can likely be returned from multiple functions, you'll end up with a set of errors returned from a set of functions which have the same error type. It is possible, but not necessary, for that set of functions to match with module boundaries. So one error type per module is sometimes OK, but I wouldn't recommend it as a general rule.

Furthermore, at different levels of abstraction, the right level of detail for errors will change (we'll discuss this a bit more below). So, the right level of detail for error enums deep inside your implementation is probably different to the right level in the API.

For example,

```rust
pub struct MessageError {
    kind: MessageErrorKind,
    request: Request,
}

pub enum MessageErrorKind {
    Network(IoError),
    Parse(Option<ProtocolVersion>),
}

enum NetworkError {
    NotConnected,
    ConnectionReset,
    // ...
}

enum ParseError {
    Malformed(MalformedRespError),
    MissingHeader(String),
    UnknownField(String),
}

struct MalformedRespError {
    // ...
}
```

In this example (over-engineered for such a small example, but imagine it as part of a larger program), `NetworkError` and `ParseError` are used in the implementation and are designed to contain maximal information for recovery. At the API boundary (if we can't recover), these are converted into a `MessageError`. This has is a struct since all errors will contain the request which caused the error, however, it still follows the 'enum taxonomy' approach rather than the 'single struct' approach since the error kinds hold different data. There should be enough information here for users of the API to recover or to report errors to the user.

If designing based on how an error might be handled, you will want a different variant for each different recovery strategy. You might want a different enum for each recovery point, or you might just want a single enum per module or crate. For example,

```rust
pub enum Retryable {
    Network(IoError, Request),
    Malformed(MalformedRespError),
    NotRetryable(Box<dyn Error>),
}

pub enum Reportable {
    MissingHeader(String),
    UnknownField(String),
    RetryFailed(Retryable),
    NotReportable(Box<dyn Error>),
}

pub struct MalformedRespError {
    // ...
}
```

### Nested errors

One thing to be aware of in this approach is nesting of errors. It is fairly common to have variants which simply contain a source error, with a simple `From` impl to convert one error to another. This is made ultra-ergonomic by the `?` operator and attributes in the error handling libraries such as thiserror. I believe this idiom is *very* overused, to the point where you should consider it an anti-pattern unless you can prove otherwise for the specific case.

In most cases where you create an error out of another error, there is additional context that is likely to be useful when recovering or debugging. If the error is simply nested, then this extra context is missed. Sometimes, if it seems like there is no extra useful context, then it might be a signal that the source error should be a variant of the other error, rather than nested in it. Alternatively, it might be that the level of abstraction should be changed.

For example, you might have a `std::io::Error` nested in your enum as `MyError::Io(std::io::Error)`. It might be that there is more context to include, such as the action, filename, or request which triggered the IO error. In this case, `std::io::Error` comes from another crate, so it cannot be inlined. It might be that the level of abstraction should change, e.g., if the io error occurred when loading a config file from disk, it could be converted into a `MyError::Config(Path)`.

To add more context to an error, use `.map_err(|e| ...)?` rather than just `?`. To change the abstraction level, you might be able to use a custom `From` impl, or you might need to use `map_err` with an explicit conversion method.

Often, this pattern occurs when primarily considering logging errors rather than using errors for recovery. IMO, it is usually better to have fine-grained logging where the error occurs, rather than relying on logging errors at some central point. See the [error handling](TODO) section for more discussion.

Another issue of this approach is that it can lead to cycles of error types, e.g.,

```rust
pub enum A {
    // ...
    B(Box<B>),
}

pub enum B {
    Foo,
    A(A),
    Other(Box<dyn Error>),
    // ...
}
```

Here you might have a `B` inside an `A` inside a `B` or vice versa, or more deeply nested. When it comes to error recovery, and you pattern match on a `B` to find a `B::Foo`, you might miss some instances because the `B::Foo` is hidden inside a `B::A` variant. It could also be hidden inside the `B::Other` variant, either as a `B` or an `A`. This will make error recovery messy where it shouldn't be.

The solution is to aggressively normalize errors in the `From` impls. However, the automatically generated impls won't do this for you, so that means writing them by hand.

A corollary to this thinking on nested errors, is that `Error::source` is not a very useful method (IMO, it is only useful for logging errors and debugging).

## Single struct style

In this style, you use a single error struct. Each error stores mostly the same data and this is stored separately to the kind of error. Though you will likely still use an enum to represent the kind of error, this will likely be a C-like enum without embedded data. E.g.,

```rust
pub struct MyError {
    kind: ErrorKind,
    source: Box<dyn Error>,
    message: Option<String>,
    request: Option<Request>,
}

pub enum ErrorKind {
    Network,
    MalformedResponse,
    MissingHeader,
    UnknownField,
    // This lets us handle any kind of error without adding new error kinds.
    Other,
}
```

This style is less common in the ecosystem. A prominent example is [`std::io::Error`](https://doc.rust-lang.org/nightly/std/io/struct.Error.html), note that this error type is further optimised by using a packed representation on some platforms.

The advantages of the single struct style are:

* it scales better when there are lots of errors (there is not a linear relationship between the number of errors and the number of error types),
* logging errors is simple,
* catching errors with downcasting is not too bad, and familiar from the trait object approach to error handling,
* it is easily customisable at the error site - you can add custom context without adding (or changing) an error kind,
* it requires less up-front design.

You have a choice about what information to include in your error struct. Some generic information you might like to include are a backtrace for the error site, an optional source error, some error code or other domain-specific way to identify the error, and/or some representation of the program state (this last one is very project-specific). If you include a source error, then you might face similar issues to those described in the previous section on error nesting.

You might also include an error message created at the error site. This is convenient and flexible, but has limitations in terms of reporting - it means the message cannot be customized (e.g., a GUI might want to format an error message very differently to a CLI) or localized.

You will probably want to include the kind of error and use an `ErrorKind` enum for this. Error recovery is mostly impossible otherwise. You then have to decide how detailed to make the error kinds. More detail can help recovery but means more variants to keep track of and too much detail may be of little use (especially if your error struct has a pre-formatted message and/or other information).


## Trait objects

You may choose to you use a trait object as your error type, `Box<dyn Error>` is common. In this case, you still need concrete error types and the design decisions discussed above still apply. The difference is that at some point in your code, you convert the concrete error types into the trait object type. You could do this at the API boundary of your crate or at the source of the error or anywhere in between.

There are some advantages of using a trait object error type:

* assuming you are using different error types for your module internals and your API, it saves you having a second set of error types for the API,
* it is statically much simpler to have a single type rather than a complex taxonomy of types,
* assuming users will not do 'deep' recovery, using the errors can be simpler too,
  - in particular, logging is more uniform and simpler,
* 'dynamic typing' of errors is more flexible and makes backwards compatibility and stability easier.

### The error type

Since `dyn Trait` types do not have a known size, they must be wrapped in a pointer. Most cases can use `Box` since that is most convenient. For systems with no allocator, or where micro-optimization of the error path is required, you can dedicate some static memory to holding an error and use a `&'static` reference.

The most common trait to use for the trait object is the `Error` trait from [the standard library](https://doc.rust-lang.org/nightly/core/error/trait.Error.html). This is part of core (c.f., std) so can be used in no-std environments. An alternative is to use the `Error` trait from [Anyhow](https://github.com/dtolnay/anyhow), an error handling crate. This has the advantage that you don't need to use unstable features of the standard library. It has the same features as the standard library trait, but with a slightly different API.

You could use your own error trait type. For ease of working with other code, you probably want it to have `std::error::Error` as a super-trait. However, I'm not aware of any advantage of doing this over using concrete error types.

### Exposing additional context

As described in [the Result and Error section](../rust-errors/result-and-error.md), you can dynamically provide additional context from an Error object. This should be well-documented in your API docs, including whether users can rely on the stability of the additional context.

Useful context to provide may include:

* a backtrace,
* pre-formatted messages as strings for logging or reporting to users (though not that this precludes localization or other customisation of the message),
* error numbers or other domain-specific information.

If using dynamic trait object errors, the errors are only useful for coarse-grained recovery, logging, or for direct reporting to users. If you want to enable fine-grained recovery, you are probably better off using concrete error types, rather than adding loads of context to error trait objects.

### Eliminating concrete error types

You can't really eliminate all concrete error types; due to Rust's nature, you have to have a concrete type which is abstracted into the trait object. However, you *can* avoid having to define your own concrete types in every module. While this is a reasonable approach, I think it is probably only the best approach in some very limited circumstances.

How to do this depends on how much information you need to get from the `Error` trait object. If you want to provide some additional context, you'll need somewhere to store that context or the data required to create it. Some approaches:

* use an error number (since the integer types don't implement `Error`, you will need to wrap the number in a newtype struct, e.g., `pub struct ErrNum(u32);`),
* use a string (again, you'll need a newtype, or if you use Anyhow you can make use of the `anyhow!`/`bail!` macros),
* use a the single struct style described above with a very simple structure.


## Picking an approach

First of all, be aware that you are not making *one* choice per project. Different modules within your project can take different approaches, and you may want to take a different approach with the errors in your API vs those in the implementation. As well as different approaches in different parts of your project, you can mix approaches in one place, e.g., using concrete errors for some functions and trait object errors for others, or using a mix of enum style and single struct style errors.

Ultimately the style of error type which is best depends on what kind of error handling you'll do with them (or expect your users to do with them, if you're writing a library). For error recovery at (or very close) to the error site, the error type doesn't matter much. If you are going to do fine-grained recovery at some distance from where the error is thrown (e.g., in a different module or crate), then the enum style is probably best. If you have a lot of errors in a library (and/or all errors are rather similar), or you expect the user to only do coarse-grained recovery (e.g., retry or communicate failure to the user), then the single struct style might be better. Where there can be effectively no recovery (or recovery is rare), and the intention is only to log the errors (or report the errors to a developer in some other way) then trait objects might be best.

Common advice is to use concrete error types for libraries (because they give the user maximum flexibility) and trait object errors in applications (for simplicity). However, I think this is an over-simplification. The advice for libraries is good unless you are sure the user won't want to recover from errors (and note that recovery is possible using downcasting, it's just easier if there is more information). For applications, it might make sense to use concrete types to make recovery easier. For many applications, you're more likely to be able to recover from locally produced errors rather than those from upstream libraries. Being closer to the user also makes interactive recovery easier.

For large, sophisticated applications, you probably want to use the enum style of concrete error, since this permits localisation (and other customisation) of error messages and helps with the modularity of APIs.


## General advice

### Naming

Avoid over-using `Error` in names, it is easy to end up with it repeating endlessly. You might want to use it in your type names (unless it is clear from context), but you almost certainly don't need it in variant names. E.g., `MyError::Io` is better than `MyError::IoError`.

It is fine to overload the standard library names like `Result` and `Error`, you can always use an explicit prefix to name the std ones, but you'll rarely need to.

Avoid having an `errors` module in your crate. Keep your errors in the module they serve and consider having different errors in different modules. Using an `errors` module encourages thinking of errors as a per-project thing, rather than tailoring their level of detail to what is most appropriate.

### Stability

Your errors are part of your API and therefore must be considered when thinking about stability (e.g., how to bump the semver version). If you have dynamically typed aspects to your errors, then you need to decide (and document) whether these are part of your API or not (e.g., if you use a trait object error and permit downcasting to a concrete type, can the user rely on the concrete type remaining the same? What about provided additional context?).

For error enums, you probably should mark all your enums as `#[non_exhaustive]` so that you can add variants backwards compatibly.

### Converting errors at API boundaries

It often makes sense to have different error types in your API to those in the implementation. That means that the latter must be converted to the former at API boundaries. Fortunately, this is usually ergonomic in Rust due to the implicit conversion in the `?` operator. You'll just need to implement the `From` trait for the types used in the API (or `Into` for the concrete types, if that is not possible).

If you'll be converting error types at boundaries, you'll be converting to a more abstract (or at least less detailed) representation. That might mean converting to a trait object, or it might be a more suitable set of concrete types. If you're using trait objects internally, you probably don't need to convert, although you might change the trait used. As discussed above, you should choose the types most appropriate to your API. In particular, you should consider what kind of error recovery is possible (or reasonable) outside an abstraction boundary.

Should you use the internal error as a 'source' error of the API error? Probably not, after all the internal error is part of your internals and not the API definition and therefore you probably don't want to expose it. If the expectation is that the only kind of error handling that will be done is to log the errors for debugging or crash reporting, etc. then it might make sense to include the source error. In this case, you should document that the source error is not part of the stable API and should not be used for error handling or relied upon (be prepared for users to rely on it anyway).
