# Thinking about errors

An error is an event where something goes wrong in a program. However, there are many different ways things can go wrong, and many different kinds of 'wrong'. The details of an error event dictate how it should be represented and handled. In this section, we'll cover many different ways to categorise and characterise errors.

I'm not sure if the terminology I use here is all standard, I've certainly made some up.

## Internal vs external errors

Consider a JSON parser. It opens a file, reads the file, and attempts to parse the data as JSON into a data structure. There are many things which could go wrong. The user might supply a filename which does not exist on disk, trying to open the file will fail. If the filename is correct, when the program attempts to read the file, there might be an operating system error which causes the read to fail. If reading succeeds, the JSON might be malformed, so it cannot be parsed correctly. If the file is correct, but there is a bug in the program, some arithmetic operation may overflow.

It is useful to categorise these errors as 'external', where the input is incorrect, and 'internal', where the input is correct but something unexpected happens.

An incorrect filename or malformed JSON data are external errors. These are errors which the program should expect to happen sometimes.

Failure to read the file, or arithmetic overflow are internal errors. These are not due to user input, but due to a problem with the program's environment or the program itself. These errors are unexpected, to varying extents. Bugs in the program are a kind of internal error, these may produce an error which the program itself can detect, or may be a silent error.

In this example, the input was user input, but in general, an external error might be caused by input which comes from another part of the program. From the perspective of a library, malformed input from the client program would cause an external error.

Note that although I've said some errors are expected/unexpected there is not a perfect correlation with internal/external. A library which states that the client is responsible for data integrity might treat malformed input as an unexpected external error. A failed network connection on a mobile device is an expected internal error.

The split between internal and external is not clean, since what constitutes a program is often not clearly defined. Consider a distributed system (or even a system with multiple processes), errors caused by other nodes/processes (or due to interactions of nodes, such as a transaction error) might be considered internal from the perspective of the whole system and external from the perspective of a single node/process.

One can also extend this thinking within program, making a distinction between errors which occur in the current module, compared with errors which occur in one module and are handled in a different one.

## Expected/unexpected spectrum of errors

Some errors happen in the normal operation of a program and some are very rare. I believe treating this as a binary is incorrect and we should think about a spectrum of likelihood of an error occurring.

At one extreme, some errors are impossible, modulo cosmic rays, CPU bugs, etc. Some errors are possible, but should never happen and will only happen if there is a bug in the program. Then there are errors which are expected to happen sometimes, with different frequencies, e.g., file system errors are fairly unlikely to happen, network errors are much more likely, but still rare in a server, but common-ish on a mobile device. At the other extreme, one can write code which always triggers an error condition (and arguably this is not a bug if the error is handled, but it may be considered poor programming practice).

## Recoverable vs unrecoverable

Can a program recover when an error occurs? Documentation for Rust, often starts with a split between recoverable and unrecoverable errors. However, I think this is a more complicated question. Whether a program can recover from an error depends on what we mean by recovering (and what we mean by 'program' - is it recovery if an observer process restarts the erroring process?), it may depend on the program state before and after the error, it may depend on where we try to recover (it might be impossible to recover where the error occurs, but possible further up the call stack, or vice versa), it may depend on what context is available with the error. Even if it is technically possible to recover, if that would take an unreasonable amount of effort or risk, then it might be better to consider the error as unrecoverable.

There is also the question of intention. An error might be technically recoverable, but a library might want to treat it as unrecoverable for some reason.

So, I think there is a whole spectrum of recoverability which we can consider for errors and the context where the error occurs.

## Multiple errors

Errors combine in all sorts of ways:

* One error might cause another error as it is propagated or during recovery.
* An error might be deliberately converted into a different kind of error.
* An error might re-occur after recovery, or even after a 'successful' recovery, might cause a different error.
* Errors might be collatable, i.e, after an error occurs and execution continues, other errors are collected and presented to the user all at once.

## Execution environment

This one is a bit different to the others. It is important to consider how a program is executing when thinking about errors. We might want to treat errors differently when testing or debugging code, c.f., when the code is in production. We might also treat errors differently if our program is embedded in another (either as a library, a plugin, or some kind application-in-an-application-server scenario). Likewise, if our program is the OS, errors will be treated differently to if it is an application running on top of the OS. A program might also be inside some kind of VM or container with its own error handling facilities, or it might have some kind of supervisor or observer program.
