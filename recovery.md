# Error recovery

This chapter covers recovering from errors. That is, what a program can do when an error occurs, because producing an error (even with perfect structure and information) is only half the story of error handling.

What can a caller do if a callee function returns an error? Handling an error means one of the following:

- Propagate it to the next caller.
- Recover from the error and carry on.
- Log it.
- Report it to the end user.
- Terminate the thread or process.

This chapter covers how to choose which to do and how it can be done.

It is worth bearing in mind that there is a cost to error handling. The more time you spend implementing error handling, the less time you have for implementing other things. Having multiple paths or states for the program can increase complexity, and sometimes have an impact on performance. We won't consider that trade-off in the rest of this chapter, but it is worth keeping in mind that more error recovery is not always better error recovery.


## Error propagation

Error propagation means that in response to an error, a function returns an error to its caller (with or without converting it to a different error type). In Rust, this can be done with `?` (which will do an implicit conversion using `From::from`), or manually by using `return`. The latter case is often part of a pattern where an error is `match`ed and propagated in some cases and handled in another way in other cases.

An error should be propagated to where it can be handled most effectively. Code close to where the error occurred is likely to have more specific context for handling the error (e.g., the values in local variables which are lost when returning from a function), and have better options for recovery (e.g., retrying an operation is usually easier if you know exactly what operation went wrong). On the other hand, code further from the error site might be a more appropriate place to handle an error (e.g., because it's application code rather than library code), may have more options for recovery (e.g., is able to access IO resources for interacting with an end user), or might have more non-specific context (e.g., knowing what operations were executed before the one which caused an error). It might also be useful to log or report information which is only available further from the error site (e.g., if a network connection fails, it might be useful to know what the connection was for and what should have been sent).

Some people propagate errors to facilitate code reuse. In this pattern, all errors are propagated back to the `main` function (or close to it) and there is a single point where all errors are handled. In my opinion, this is an anti-pattern and should be avoided. Error handling code can be reused simply by calling shared functions from the most appropriate location. The advantage of the centralized error handling pattern over simple shared functions is only deduplicating the calls to shared functions, and that has minimal benefit. On the other hand, handling errors at the most appropriate place usually has huge benefits in terms of having better context and better recovery options.


## Classifying errors

### Recoverable and unrecoverable errors

A recoverable error is one which is possible to recover from (not necessarily that you *should* recover from it), and an unrecoverable error is one which is impossible to recover from in a way which leaves the program (or wider system) in a consistent state. We usually (but not always) describe errors which can only be safely recovered from by restarting the program (or the system) as unrecoverable.

Practically, there is quite a gray area between recoverable and unrecoverable. Many errors are sometimes recoverable, depending on the context. Some errors are possibly recoverable, but doing so is difficult, unlikely to succeed, takes a long time, or is high risk. An unrecoverable error might require a simple restart of the thread, process, or node, recalculating or refreshing persistent state, resetting to a 'known good' state, or require intervention by an end-user, administrator, or supervisor process.

Identifying whether an error is recoverable or unrecoverable is important, but understanding the exact nature of potential recovery (or lack of recovery) is essential for good error handling.


### Expected and unexpected errors

Another way to classify errors is as expected or unexpected. Nearly all errors are unexpected to some degree. The only kind that aren't are errors in user input (e.g., syntax errors in a compiler). These should nearly always be handled as part of the success path, rather than as a program error. However, some errors are certainly more expected than others.

Here are some points on the spectrum of expected-ness, from most expected to most unexpected:

- Some are expected as part of normal operation and must be handled as part of normal operation. E.g., failure to commit a transaction, incorrect password, expired authorization, or a dropped packet.
- Some are expected, rare, and must be handled, but recovery might be an unusual state for the system. E.g., a node in a distributed system going offline, ongoing network connectivity issues, or integer overflow (which might actually be frequent and expected, or practically impossible, depending on the context).
- Some are unexpected, but happen frequently enough that they must be properly handled for a system to be considered robust. E.g., local IO failing, or running out of system resources.
- Some are unexpected and should only happen if the system is misused or attacked. E.g., a badly formatted file or data (e.g., string data is not utf8, expected length of some data doesn't match real length).
- Some should be impossible. E.g., bit flips due to cosmic rays or, importantly, errors caused by bugs in the software (bugs are very much possible! It is the effect of bugs on the program that, from the program's perspective, should be impossible).

How you should handle errors is a function of how recoverable the error is, how expected the error is, and the level of robustness/resilience required of the system.


## Logging and reporting errors

We often want to log or report errors in some form. Logging is usually done as well as some other kind of handling. If and how errors are logged depends on whether the error was expected or not, and if the program recovered or not. Unexpected errors or errors where the program can't recover are likely to be logged. Expected errors that are recovered from might not be logged, or might be logged using a lower log level (e.g., `debug` rather than `error`) because they might help with debugging but not be useful for regular monitoring. Even if an error is not logged, if it can affect performance or occurrences can give some insight into operation, it might be useful to count occurrences as a metric.

In some cases we might want to log an error if we can't recover, but not if we do recover. Or perhaps log these two code paths at different log levels or with different information. That brings us to the question of where to log errors. As with the question of whether or not to propagate an error, there is a trade-off in terms of the information available. Logging an error where it occurs (or close to it) means we have specific context (perhaps even a useful code location or backtrace). Logging an error further up the call chain means we have more general context. If we want to log an error only if it can't be recovered from, then we have to do the logging at the recovery site, not the error site. It can sometimes be useful to log when an error is propagated, usually if the error is converted to a different type (which might lose some information) or if there is useful context information available at the propagation site.

Where errors are logged can affect the design of error types: if logging happens away from the error site (where the error is handled or propagated, or when it reaches `main` or similar), then the error must contain all the information that you want to log. If an error is logged at the error site, then that information can be used directly and is not required to be in the error type.

Reporting may be similar to logging in that it is mainly informational and communicates information about an error to something or someone outside of the program. The audience is different: for logging the audience is developers or administrators, for reporting it is end-users. Logging usually does not interrupt usage and happens in the background; reporting usually interrupts the user. Therefore, the bar for when to report an error is usually much higher than for logging. Other than that, informational reporting is similar to logging. Reporting may also occur when operation can no longer continue, as some kind of exit message.

Unlike logging, reporting is part of the user interface of a program. How error reports are presented to the user might vary widely, e.g., using a GUI dialog box, using HTML, using a custom GUI (e.g., in a game), using a plain-text console, or using a fancy console with colors and line drawing etc. It is possible that an error report will be presented in different ways if the core program can be used with different interfaces (e.g., a compiler which can be used on the command line or via an IDE).

Like other parts of the UI, error reporting might need to handle internationalization and localization, accessibility, customizing messages for different users (e.g., different levels of authorization) or modes of operation, and so forth.


### My advice

My opinion is that all errors which should be logged should be logged as close to where they occur as possible. Some errors should be logged where they are converted to a different type, especially where an error is converted from a type external to your crate into an internal error type. In this case logging as part of the `From` implementation can be useful. What happens when an error is handled might also be logged (possibly in addition to when the error first occurs). The case of only logging an error if recovery fails should be handled by logging both the error site and the recovery site with enough context to link the two. If it is too noisy to log every error, then only log at the recovery site and include any required information in the error type (but only if you have to).

If you follow this advice, then you should only very rarely need to include information only needed for logging in an error type. As we'll discuss later in the guide, I think that is a good thing!

In my opinion, error reporting should be kept to a minimum. Unless a program is terminating, error reporting is rarely useful for a user, and logging is a better alternative. Error reporting on termination, or error reporting of expected errors (e.g., syntax errors from a compiler, though as mentioned earlier, these should not be treated as an error in the `Result` sense) should be considered as part of the UI design. That probably means the error should be a structured type and it is presented via some kind of view layer (to handle customization, localization, etc., as well as rendering to different forms of output). You should probably not use an error's `Display` impl for error reporting. Instead consider it a human-readable representation of an error, for an audience of developers and administrators (i.e., useful for logging rather than reporting to an end-user)[^display-mistake].


[^display-mistake]: My controversial opinion is that it was a mistake for the `Error` trait to require a `Display` implementation and should only require `Debug`. But this can get a bit philosophical about the nature of `Display` and `Debug`.


## Terminating the thread or process

When handling an error, just giving up and quitting is a (relatively) easy option. It can be a good option too in some circumstances, and a good fallback if nothing else works. However, there are still some things to consider to make it a safe option.

There are in fact two options: terminate the current thread, or terminate the whole process. Terminating the thread usually means panicking, either using `unwrap` or by explicitly using `panic` in response to some subset of errors. The important thing with panicking is ensuring that other threads can continue running; that is covered in the [panicking chapter](rust-errors/panic.md).

Terminating the whole process means you don't need to worry about other threads, but we do need to ensure that the program can be restarted and run correctly. That means ensuring that any persistent state (e.g., stored on disk, remotely (in the cloud or on another node in a distributed system), or in other processes) is consistent.

Terminating the process can be abrupt (e.g., using [`process::exit`](https://doc.rust-lang.org/stable/std/process/fn.exit.html)) or well-organized, called *graceful shutdown*. The former is much easier, but likely to give good results only if there is no persistent state, or if corrupt persistent state can be recovered on startup. Graceful shutdown requires more work (in particular, it means that the whole program has to be able to operate in a 'shutting down with an error' state where many assumptions are likely not to hold), but is better for keeping consistent persistent state.

One way to implement graceful shutdown is to terminate the current thread by panicking and ensuring that the panic is propagated to all other threads, which ensures that all destructors are called. The same issues with unwinding a single thread apply here too. Alternatively, graceful shutdown can be implemented as an explicit action within the program without using panicking or unwinding. That may or may not start by unwinding the current thread. This approach is preferred in systems with a lot to do during shutdown (e.g., communicating with other nodes) and which are able to do so fairly easily with limited resources (because of the error). It is also a good option in programs which do significant async work because Rust does not support async destructors.

If the program is user-facing, then terminating will be a poor user experience, so it should be avoided. In other cases, the program should often be restarted. That requires some kind of *supervision*. Supervision can either be in-process or out-of-process. In-process supervision means that at the highest level of a program, execution can be restarted without actually restarting the process (so this is a logical restart rather than literally restarting). Care must be taken to ensure that all state is completely reset; it's easy to miss a static variable or IO resource somewhere. The advantage of this approach is simplicity: it doesn't require setting up an environment to run the program in.

Out-of-process supervision is more common and means having some kind of supervisor process which detects when the supervised process exits and restarts it. Restart may be optional depending on how or why the supervised process terminated. The supervisor may also reset, change, or check the environment to make the success of the restart more likely.

A specific kind of out-of-process supervision is remote supervision, where the supervisor is on a physically or logically (e.g., a different VM on the same machine) remote node and restarts either the terminated process or the whole supervised node when it detects the supervised process is no longer reachable. Obviously, this is more complex, but it also allows for a replacement node to be brought up in the case of repeat failure (perhaps due to hardware failure), replacing a single node with multiple nodes (if the failing node was likely overwhelmed), shifting the node's work to other non-failing nodes rather than restarting the failing one, or handling the case of a node being unreachable but still running (e.g., due to network failure).

Whatever kind of termination or supervision is used, a significant hazard with restarting a process or thread is that the restarted process or thread quickly hits the same error and is restarted again, leading to an infinite loop of errors and restarts. This can be avoided by only retrying a limited number of times, not retrying the same thing (by changing the parameters of an operation or ensuring that a different operation is used), or both. Avoiding these loops shares a lot with retrying an operation immediately in the face of an error (described in the next section).


## Recovering from an error

This section covers recovering from an error without terminating a thread or program. There are a few tactics for such recovery:

- Ignore the error and carry on.
- Retry the operation which failed.
- Try an alternative operation.
- Sophisticated, domain-specific recovery.

Ignoring an error is possible if the failed operation was optional. It might be useful to record the error in logs or metrics to know how frequently it occurs. Implementation is usually trivial, for example by using a default value.

Retrying an operation is a broad category. If the error is likely to be intermittent and due to the environment rather than your program (e.g., a network connection error), then a simple retry might be appropriate. Otherwise, you will need to change something before retrying (because if you don't you can expect the same error again). This might be a parameter of the operation, some context of the operation, or the target of an operation. Some examples:

- An operation that uses a delta of data or cached data and can be retried with the full data. Similarly, derived data could be re-computed.
- An operation on multiple data can be retried with fewer data.
- An environment variable or config file can be changed before retrying the operation.
- The same operation can be sent to a different node in a distributed system.

You should probably limit the number of retries to avoid an infinite loop of errors and retrying. How many retries are appropriate is very domain-specific. Instead or as well, you might want to limit the time spent retrying.

You may also want to wait some time before retrying (called *back-off*). This can be a simple pause, a pause which grows with each retry (usually exponentially, the reason being that the more often an operation fails, the more likely it is to fail again, and the longer it is likely to take before an issue resolves), or either of those with some *jitter*. Jitter is a small random adjustment to the wait time so that if other systems are retrying the same action, then not all the retries arrive at the same time, potentially overwhelming the 'server' system.

Another kind of retry is to try a different operation (this might be as well as a simple retry or a retry with different data). The alternative operation is usually one which does roughly the same thing as the failing one, but with a less optimal implementation. Some examples:

- The same operation sent using a different protocol or encoding.
- A similar operation using a different API.
- A slower or more memory-heavy version of the failing operation.
- An operation with different performance trade-offs (e.g., one which is slower but uses less memory).
- A more general or more specific operation.
- An operation with less precision.
- An operation with less customization.
- An operation with less stringent requirements.

One thing you should never do is retry an operation using a less secure implementation. A system is only as secure as its weakest point, and using a less secure operation means that the weaker level of security is the level of security that must be considered normal, because it is usually possible for an attacker to create a scenario which causes an error and forces the system to use the less secure operation.

Finally there is the option of some kind of domain-specific recovery. Obviously, these strongly depend on the nature of the system. Some examples:

- Rollback or abort a transaction or action.
- Fallback to saving data locally rather than remotely (or vice versa).
- Renew authorization, authentication, or other credentials.
- Get human input (from the user or an administrator). This might be as simple as re-typing a password or be arbitrarily complex.
- Garbage-collecting older or expired data.
- Reducing detail of rendered graphics.


### My (somewhat cynical) observations of real life systems

I think that in general, engineers are too optimistic about how much work software will do to recover from errors. Outside of some systems with high requirements for resilience, failing operations might be retried (if they are likely to be intermittent, and without changing any input of the operation), otherwise errors are reported to the user and the process terminates, or logged and the system carries on. Getting a system back into a consistent state is only managed if the error can be ignored (i.e., the system is not inconsistent), or in high-resilience systems.

This isn't necessarily bad. If it's fine for a system to terminate or carry on, then putting more effort into error recovery would be a bad use of resources. But I think it is important to understand how your system is likely to be used in real life, and not over-engineer your error handling to match some hypothetical ideal of error recovery.


## Summary of error recovery advice

- If necessary, inspect an error to decide on the appropriate action (e.g., by `match`ing an error enum).
- Propagate an error to the appropriate place to handle it. That might be a different place depending on details of the error. Where to handle an error is a trade-off between specific and general context. Don't centralize error handling or propagate by default - error recovery is likely to be more feasible closer to the error site.
- Log the error at the error site and/or where it is decided that recovery is not possible.
- If recovering, ensure all program state is consistent.
- Retry or use some domain-specific recovery if possible. Consider the timing of retries and how many times an operation should be retried.
- Abort or panic if necessary, ensuring that any persistent state is left consistent (or can be fixed on restart). Log or report the action.
