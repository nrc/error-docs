# Testing

This chapter will cover the intersection of error handling and testing. We'll cover how to handle errors which occur in tests, both where an error should indicate test failure and where we want to test that an error does occur.

You should ensure that your project has good test coverage of error paths as well as testing the success path. This might mean that errors are propagated correctly, that your code recovers from errors in the right way, or that errors are reported in a certain way. It is particularly important to test error recovery, since this can lead to program states (often many complex states) which are not encountered in normal execution and might occur only rarely when the program is used.

## Handling errors in tests

In this section we'll assume a test should pass if there is no error and the test should fail if there is any error.

A basic Rust unit test fails if the function panics and passes if it does not. So, if you use only panics for error handling, then things are very easy! However, if (more likely) you have at least some `Result`s to handle, you have to do a bit more work. The traditional approach is to panic if there is an error, using `unwrap`. Since 2018, you can return a `Result` from a test. In this case the test fails if the test returns an error or panics, so you can use the `?` operator to fail a test on error. To do this simply add `-> Result<(), E>` (for any error type `E`) to the signature of the test.

## Testing the kind of errors

To properly test your program, you should test the errors which your functions throw. Exactly what to test is an interesting question: for internal functions you should test what is needed, this might mean just that any error is thrown (for certain inputs). If you rely on certain errors being thrown (e.g., for recovery) then you should test that a specific error is thrown or that the error contains correct information. Likewise, for testing public function, you should test what your API guarantees (do you guarantee certain error types under some circumstances? Or only that *an* error is thrown?). 

To require that a test panics, you can use the `#[should_panic]` attribute. You can use an `expected` parameter to test for a particular message when panicking, e.g., `#[should_panic(expected = "internal compiler error")]`.

You could use `#[should_panic]` with `unwrap` to test that an error is thrown, but it is a bad idea: you could easily pass the test by some other part of the code panicking before the error should occur.

The better way to test for the presence of an error is to `assert` `Result::is_err` is true. If there are more properties of the error to check, then you can use `Result::is_err_and` for simple checks or `Result::unwrap_err` where there is more code. For example,

```rust
#[test]
fn test_for_errors() {
    assert!(should_throw_any_error().is_err());
    assert!(should_throw_not_found().is_err_and(|e| e.kind() == ErrorKind::NotFound));

    let e = should_throw_error_foo().unwrap_err();
    assert_eq!(e.kind(), ErrorKind::Foo);
    assert_eq!(e.err_no(), 42);
    assert_eq!(e.inner().kind(), ErrorKind::NotFound);
}
```

Exactly how to test properties of the error thrown will depend on what kind of error types you are using and what guarantees you are testing. You may be able to call methods on the error and assert properties of the results (as shown in the example above). If you want to test the type of the error then you'll need to either downcast (if using trait object error types) or match (or use `if let`; if using enum error types). In the latter case, the `assert_matches` macro can be useful - you can test that an error is thrown and the exact type of the error in one go, e.g., `assert_mactches!(should_throw_not_found(), Err(MyError::NotFound));`.


## Testing error recovery and handling

It is important to test the error handling/recovery paths, however, with idiomatic error handling this is not trivial because it is not idiomatic to pass `Result`s into a function, but rather to test the result and only pass in a value if the result is `Ok`. This means that the only way to inject errors when testing is to have functions return an error. Unless you have functions which take a closure, then the best way to do this is by using mock objects.

Mocking in Rust is usually ad hoc and lightweight. The common pattern is that you have a generic function or function which takes a trait object and you pass a custom mock object which implements the required trait bounds. For example,

```rust
// Function to test
fn process_something(thing: impl Thing) -> Result<i32, MyError> {
    // ...
}

trait Thing {
    fn process(&self, processor: Proc) -> Result<f32, MyError>;
}

#[cfg(test)]
mod test {
    use super::*;

    // Mock object
    struct AlwaysZero;

    impl Thing  for AlwaysZero {
        fn process(&self, processor: Proc) -> Result<f32, MyError> {
            Ok(0.0)
        }
    }

    // Mock object
    struct AlwaysErr;

    impl Thing for AlwaysErr {
        fn process(&self, processor: Proc) -> Result<f32, MyError> {
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
        assert_matches!(process_something(AlwaysErr), Err(MyError::ProcessError(_)));
        Ok(())
    }
}
```

For more sophisticated mocking, there are several mocking libraries available. [Mockall](TODO) is the most popular. These can save a lot of boilerplate, especially when you need mock objects to do more than always return a single error.


## Testing error reporting

For applications with sophisticated error reporting (e.g., a compiler), you'll want to test that error reporting output, ensuring that the messages which are expected and other data such as the input which caused the error, information to locate that input, error numbers, etc.

How to test this will vary by application, you might be able to unit test the output or you might prefer to use integration tests. You'll want to implement some kind of framework to help these tests so that you're not doing loads of repetetive string comparisons. It's important not to test too much or your test suite will be fragile. An example of such tests are rustc's [UI tests](https://rustc-dev-guide.rust-lang.org/tests/ui.html).

## Benchmarking

If errors can occur in real life, then you should probably consider errors when benchmarking. You might want to separately benchmark what happens in the pure success and the error cases. You might also want a benchmark where errors occur randomly with realistic (or greater than realistic) frequency. This will require some engineering which is similar to mocking, but does the real thing in some cases and mocks an error occasionally.
