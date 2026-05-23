# Testing

This chapter covers the intersection of error handling and testing. We'll describe how to handle errors which occur in tests, both where an error should indicate test failure and where we want to test that an error does occur.

You should ensure that your project has good test coverage of error paths as well as testing the success path. This might mean that errors are propagated correctly, that your code recovers from errors in the right way, or that errors are reported in a certain way. It is particularly important to test error recovery, since this can lead to program states (often many complex states) which are not encountered in normal execution, and might occur only rarely when the program is used.


## Language support

A basic Rust unit test fails if the function panics and passes if it does not. So, if you use only panics for error handling, then things are very easy! However, if (more likely) you have at least some `Result`s to handle, you have to do a bit more work. Since the 2018 edition, you can return a `Result` from a test. In this case the test fails if the test returns an error or panics, so you can use the `?` operator to fail a test on error.

Another useful tool is the `#[should_panic]` attribute on tests. This makes a test pass if it panics and fails if it doesn't. However, you won't know what caused the panic - it could be part of the program which shouldn't have panicked. You can use an `expected` parameter to test for a particular message when panicking, e.g., `#[should_panic(expected = "internal compiler error")]`[^rstest]. But testing the `Result`s directly gives you even more precision, so is a better strategy than `unwrap`ing if you are starting with `Result`.

There are various methods on `Result` that are useful in tests. You can use `is_err` or `is_ok` with the `assert` macro to test success/failure, or `is_err_and` to assert something about the error. `inspect_err` and `unwrap_err` (and `unwrap` of course) are sometimes useful. Simply `match`ing on a `Result`, or using the `assert_matches` (nightly only for now, but expected to stabilize soon) or `matches` macros is a powerful but sometimes verbose approach. One advantage of matching the `Result` or `Error` is that you can assert a particular error variant is present without testing its payload values.

For example,

```rust
#[test]
fn test_for_errors() {
    assert!(should_return_any_error().is_err());
    assert!(should_return_not_found().is_err_and(|e| e.kind() == ErrorKind::NotFound));

    let e = should_return_error_foo().unwrap_err();
    assert_eq!(e.kind(), ErrorKind::Foo);
    assert_eq!(e.err_no(), 42);
    assert_eq!(e.inner().kind(), ErrorKind::NotFound);
}
```

Even though `Display` is a super-trait of `Error`, I'd recommend against converting errors to strings and asserting on the contents. This makes for very fragile tests which break when superficial details of the error message change, and inaccuracy if different errors (or errors carrying different values) have the same string representation.

[^rstest]: if you use the [rstest](https://docs.rs/rstest/latest/rstest/) crate for testing, the expectations can be set on a per-case basis.


## Testing error recovery and handling

It is important to test the error handling/recovery paths, however, with idiomatic error handling this is not always easy because it is not idiomatic to pass `Result`s into a function, but rather to check the result of a function and handle an error if necessary. This means that the only way to inject errors when testing is to have functions return an error. Unless you have functions which take a function as an argument, then the best way to do this is by using mock objects.

Mocking in Rust is usually ad hoc and lightweight (in contrast to other languages which often have sophisticated frameworks or libraries). The common pattern is that you have a generic function, or function which takes a trait object, and a mock object which implements the required trait bounds. For example,

```rust
// Function to test
fn process_something(thing: impl Thing) -> Result<i32, MyError> {
    thing.process()
}

trait Thing {
    fn process(&self) -> Result<f32, MyError>;
}

#[cfg(test)]
mod test {
    use super::*;

    // Mock object
    struct AlwaysZero;

    impl Thing for AlwaysZero {
        fn process(&self) -> Result<f32, MyError> {
            Ok(0.0)
        }
    }

    // Mock object
    struct AlwaysErr;

    impl Thing for AlwaysErr {
        fn process(&self) -> Result<f32, MyError> {
            Err(MyError::IoError)
        }
    }

    #[test]
    fn process_zero() -> Result<(), MyError> {
        assert_eq!(process_something(AlwaysZero)?, 0);
        Ok(())
    }

    #[test]
    fn process_err() -> Result<(), MyError> {
        // Note that we test the specific error returned but not the contents of
        // that error. This should match the guarantees/expectations of `process_something`.
        assert!(matches!(process_something(AlwaysErr), Err(MyError::ProcessError(_))));
        Ok(())
    }
}
```

For more sophisticated mocking, there are several mocking libraries available. [Mockall](https://docs.rs/mockall/latest/mockall/) is popular. Mockall automatically creates mock implementations of traits (as well as some functions and structs) and supports quite intricate constraints on the expected calls as well as returning custom values. That is especially useful when you want to do more than always return a single error.

Property testing might also be useful. It can be used to test error paths with random input.


## Testing error reporting

For applications with sophisticated error reporting, you'll want to test that error reporting output. Ensuring that the messages and other data (such as the input which caused the error, information to locate that input, error numbers, etc.) are as expected.

If you have custom error reporting, then you should unit-test it by supplying errors and checking the output of reporting. If you're using a third-party error reporting crate, you don't need to do this; it's their job!

In either case, you should have integration tests. For anything larger than a toy, you'll want to implement some kind of framework for these tests so that you're not doing lots of repetitive string comparisons. It's important not to test too much or your test suite will be fragile (e.g., only test one error per test, don't make assumptions about other output, try to avoid testing inconsequential details or the implementation). An example of such tests is rustc's [UI tests](https://rustc-dev-guide.rust-lang.org/tests/ui.html).


## Benchmarking

If errors can occur in real life, then you should consider errors when benchmarking. Consider a new distributed system which has good performance in testing and is tested to recover correctly when an error occurs (e.g., a node goes down). However, when the system is deployed into production with much greater load than in testing, error recovery might cause the system to slow down enough that buffers fill up, messages are dropped, and the system gets into an inconsistent state. Or perhaps although a system recovers correctly from an error, it causes slowdown, not noticed in testing because the error rate is low. However, at scale there are enough errors (and recoveries) that it significantly affects throughput or latency.

You might separately benchmark what happens in the pure success and the error cases. You might also have a benchmark where errors occur randomly with realistic (or greater than realistic) frequency. This will require some engineering which is similar to mocking, but does the real thing in some cases and mocks an error occasionally.
