# The errors ecosystem

## Good crates

The following crates are commonly used and recommended.

### ThisError

A powerful `derive(Error)` [macro](https://github.com/dtolnay/thiserror). Implements `Error` and `Display` for your error types, including convenient syntax for source errors, backtraces, and conversions.

### Anyhow

A [crate](https://github.com/dtolnay/anyhow) for working with trait object error types. Extends the features of the standard library's `Error` trait. However, most of Anyhow's features have been added to std, so you might not need Anyhow any more (it does have the advantage that it doesn't require unstable features, so can be used with a stable toolchain).

### Snafu

[Supports](https://github.com/shepmaster/snafu) deriving error handling functionality for error types (including the single struct style, not just enum style errors), macros for throwing errors, and using strings as errors.

### Error reporting crates

These crates are for reporting errors to the user in pretty ways. They are particularly useful for reporting errors in input text (e.g., source code) and are somewhat inspired by rustc's error reporting style.

[Eyre](https://github.com/yaahc/eyre) is a fork of Anyhow with enhanced reporting facilities. The following crates are just for reporting and work with other error libraries if required:

* [miette](https://github.com/zkat/miette)
* [ariadne](https://github.com/zesterer/ariadne)
* [codespan](https://github.com/brendanzab/codespan)

## Historic crates

As Rust's error handling story has evolved, many error handling libraries have come and gone. The following were influential in their time, but there are now better alternatives (sometimes including just the support in std). You might still see these crates used in historic documentation, but we wouldn't recommend using them any more.

* [error-chain](https://github.com/rust-lang-deprecated/error-chain) (support is now in std)
* [failure](https://github.com/rust-lang-deprecated/failure) (support is now in std)
* [fehler](https://github.com/withoutboats/fehler) [blog post](https://without.boats/blog/failure-to-fehler/) (an experiment which is no longer maintained)
* [err-derive](https://gitlab.com/torkleyy/err-derive) (support is built in to the recommended crates above)
