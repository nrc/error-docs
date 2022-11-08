# Error handling

This chapter will discuss what to actually do with your errors.

When your code encounters an error, you have a few choices:

* recover from the error,
* propagate the error,
* crash.

Recovery is the most interesting option and most of this section will be spent on this. If you don't recover, then you'll probably propagate the error. This is pretty straightforward, the only choice is whether to convert the error into a different one or propagate it as is. We'll discuss this more in the following chapter on error types.

Crashing can mean panicking the thread or aborting the process. The former might cause the latter, but that doesn't usually matter. Whether you abort or panic depends on the architecture of your program and the severity of the error. If the program is designed to cope with panicking threads, and the blast radius of the error is limited to a single thread, then panicking is viable and probably the better option. If your program cannot handle threads panicking, or if the error means that multiple threads will be in an inconsistent state, then aborting is probably better.

Crashing may be a kind of recovery (if just a thread crashes and the program continues, or if the program runs in an environment where crashing is expected and the program can be restarted), but usually it is the least desirable response to an error. Even for prototype programs, you are probably better to propagate errors to the top level rather than crashing, since it is easier to convert that to a proper error handling system (and prototypes turn into production code more often than anyone would like).

## Error handling strategy

Your project needs an error handling strategy. I consider this an architectural issue and should be decided in the early stages of planning, since it affects the API and implementation designs and can be difficult to change later in development.

Your strategy should include expectations for robustness (for example, how high priority is it that errors do not surface to the user? Should correctness, performance, or robustness be prioritised? What situations can we assume will never happen?), assumptions about the environment in which the code will run (in particular can errors/crashes be handled externally? Is the program run in a container or VM, or otherwise supervised? Can we tolerate restarting the program? How often?), do errors need to be propagated or reported to other components of the system? Requirements for logging, telemetry, or other reporting, *who* should handle errors (e.g., for a library, which kinds of error should be handled internally and which should the client handle), and how error/recovery states will be tested.

Whether your project is a library or an application, and if an application whether it stands alone or is a component in a larger system, will have a huge effect on your error handling strategy. Most obviously, if you are a stand-alone application then there is nobody else to handle errors! In an application you will have more certainty about the requirements of the system, whereas for a library you will likely need to provide for more flexibility.

Having identified the requirements of your strategy, this will inform how you represent errors as types (discussed below and in the [next chapter](error-type-design.md)), where and how you will recover from errors, and the information your error types must carry (discussed in the following sections).

## Result or panic?

Most programs should use `Result` for most of their error handling.

You might think that for a very quick prototype or script, you don't need error handling and unwrapping and panicking are fine. However, using `?` is usually more ergonomic than `unwrap` and these tiny programs have a habit of growing into larger ones, and converting panicking code into code with proper error handling is a huge chore. Much better to have a very simple error type (even just a string); changing the error type is much easier.

You should probably use panics to surface bugs, i.e., in situations which should be impossible. This is easier said than done. For example, consider an integer arithmetic overflow. This might be impossible, but it might also be possible given enough time or the right combination of user input. So even classes of error which usually cause panics are likely only to be best represented as a panic in some circumstances.

When designing an API, either public or private, it is generally better for a function to return a `Result` and let the caller decide to panic or not, rather than always panic on an error. It is very easy to convert a `Result` into an `Error` but the reverse is more difficult and loses information.

For how to design the error types carried by `Result`, see the [next chapter](error-type-design.md).

## Recovery

Error recovery is tightly linked to your program and to the individual error. You can't recover from a network error in the same way as a user input error (at least not all the time), and different projects (or parts of projects) will handle these errors in different ways. However, there are some general strategies to consider, and there are some ways which recovery intersects with Rust's error types.

### Techniques for recovery

Possible recovery techniques are:

* stop,
* retry,
* retry with new input,
* ignore,
* fallback.

Most of these may be accompanied by some change in program state. E.g., if you are retrying, you may increment a retry counter so that after some number of failed retries, a different technique is tried. Some programs might have much more complex 'recovery modes'.

The recovery technique is likely combined with some logging action or reporting to the user. These actions require additional information in the error type which is not needed directly for recovery. I think it is useful to think of logging or reporting as part of error recovery, rather than an alternative, since it is your code which handles the error, rather than throwing it for client code to deal with.

#### Stop

If an error occurs while performing an action, you can simply stop doing the action. This might require 'undo'ing some actions to ensure the program is in a consistent state. Sometimes this just means calling destructors, but it might mean removing data written to disk, etc. Failing to execute an action should probably be reported in some way, either back to the user or via an API to another program or part of the program.

An extreme form of stopping is crashing. You could terminate the current thread (if your program is designed to handle this happening), or the whole process. This is usually not a very responsible thing to do, but in some environments it is fine (e.g., if the process is supervised by another process or if the process is part of a distributed system where nodes can recover). Assuming that the systems can recover from such a crash, you need to consider how it will recover - if the process runs with the same input and tries the same action, will it always get the same result (i.e., crash again)?

#### Retry

If the error is due to a temporary fault or a fallible actor (such as a human user), simply retrying the action which caused the error might succeed. In other cases, you can retry with different input (e.g., by asking the user for more input, or by using a longer timeout, or asking a different server).

#### Ignore

Some errors can simply be ignored. This might happen when the action being attempted is optional in some way, or where a reasonable default value can be used instead of the result of the action.

#### Fallback

Sometimes there is an alternate path to the path which caused an error and your program can fallback to using that path. This might be as simple as using a default value instead of trying to find a better value, or it might be using an alternate library or alternate hardware (e.g., falling back to an older library if the newer library is not present on a system, or using CPU rendering rather than dedicated graphics hardware).

### Where to recover

You may choose to recover near to where the error occurs, or throw the error up the call stack and recover at some distance from the error. Both approaches have advantages and there is no nest choice.

Close to the error you are likely to have the most specific context and therefore the most information for recovery. There is also the minimum of disruption to program state. On the other hand, by throwing the error and dealing with it further from the error site, you might have more options, for example you might be able to interact with the user in a way which is impossible deep in library code. The correct recovery approach may also be clearer, especially if the error occurs in a library. Finally, if you centralize error handling to some extent, this may allow more code reuse.

### Information for recovery

If you always recover close to the error site, then your error types only need to carry information for logging/reporting (in the event that recovery fails). If you handle errors further from the error site, then your error type must carry enough information for recovery.

As errors propagate through the stack, you may want to add information to your error. This is fairly easy using the `Result::map_err` method, but you will need to account for this when designing your error types. Likewise, you may wish to remove information by making the error more abstract (especially at module or crate boundaries). This is inevitably a trade-off between providing enough information to recover, but not providing too much which might expose the internals of your module or make the API too complex.

There is also the question of how to store information in your errors. For most code, I recommend storing information structurally as data and avoiding pre-formatting data into message strings or summarizing data. Storing information as structural data lets your code's client decide how to log or report errors, enables maximal flexibility in recovery, allows localization and other customisation, and permits structural logging.


## Logging

Logging is useful for debugging and basically essential for running services in production. You will probably have a bunch of constraints around logging because of how it must integrate (technically and culturally) with other systems.

How you log should follow from your logging strategy - how you decide what to log and how to log it. That is beyond the scope of this doc. I'll cover some options, but which is appropriate will depend on your logging strategy, rather than anything I could enumerate here.

One big choice is whether to log errors where they occur or in some central-ish point or points. The former is easier since it puts less pressure on the errors to carry context. The latter makes it easier to log only unhandled (at some level) errors, and it makes it easier to change the way errors are logged. This will somewhat depend on whether you want to log all errors or only uncaught ones. In the latter case, you'll probably want to do some logging in error recovery too since this is an area where bugs (or performance issues) are more likely to be triggered.

Another choice is whether to keep data in errors structurally or pre-formatted. The latter is easier and more flexible in the sense that the data stored can be changed over time easily. However, it is less flexible in that there is only one choice for logging. In the former case, logging in different places or at different times can log different information. The information can also be formatted differently, internationalized, translated, or used to create derived information. Keeping data structurally is therefore the better choice in large or long-lived systems.

Structured logging takes the structured approach further by logging data in a structured way, rather than logging a formatted string. It is usually superior in all but the simplest scenarios.

Tracing is an approach to logging which relates log events by program flow across time. This is often much more useful than logging only point-in-time events without context. Generally, errors don't need to be aware of whether you do simple logging or tracing. However, if you're logging errors centrally, you may need to add some tracing information to errors so that when they are logged there is the relevant context for the tracing library.

### Libraries for logging

* [tracing](https://github.com/tokio-rs/tracing) is the best choice for full tracing and structured logging. It is maintained by the Tokio project, but does not require using Tokio. There is an ecosystem of integration crates and other libraries.
* [log](https://github.com/rust-lang/log) is part of the Rust project and an old standard of Rust logging. It requires a logger to actually do the logging and is therefore independent of how logs are emitted. It supports simple structured logging, but not tracing.
* [env-logger](https://github.com/env-logger-rs/env_logger/) is the most commonly used logger (used with log, above). It is simple and minimal and best suited for small projects, prototypes, etc.
* [log4rs](https://github.com/estk/log4rs) and [fern](https://github.com/daboross/fern) are more full-featured but complex loggers (which again integrate with log).
* [slog](https://github.com/slog-rs/slog) was the most popular choice for structured logging and is an alternative to log. However, it does not seem to be maintained any longer and is probably not a good choice for new systems.

I would recommend using Tracing if you need production-strength logging/tracing, or log and env-logger if you need simple logging.
