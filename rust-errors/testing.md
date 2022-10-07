# Testing

This chapter will cover the intersection of error handling and testing. We'll cover how to handle errors which occur in tests, both where an error should indicate test failure and where we want to test that an error does occur.

You should ensure that your project has good test coverage of error paths as well as testing the success path. This might mean that errors are propagated correctly, that your code recovers from errors in the right way, or that errors are reported in a certain way. It is particularly important to test error recovery, since this can lead to program states (often many complex states) which are not encountered in normal execution and might occur only rarely when the program is used.

## Handling errors in tests

On the happy path only

panics
errors -> panics
throw an error
    -> Result<..>
    TODO can the Ok type be anything or just () ?

## Testing error handling

throwing errors
    mocking
    mocking lib
testing properties of errors
    testing interface vs implementation
        what are the guarantees your API makes around errors?
    should_panic
        #[should_panic(expected = "some text")]
        don't use with unwrap to test for errors
    Result::is_err, is_err_and, unwrap_err
    assert_matches
    match, if let
    static and dynamic approach
advanced error testing - UI tests, etc.
