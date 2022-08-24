# Error type design

anything can be an error
    string errors for prototyping
    structs, enums, ...
static enums vs static ErrorKind vs dynamic style
    TODO names
    motivation for one or the other
        static
            more common
            easier matching and recovery/reporting
        dynamic
            more concise
            casting is not too hard
            good for logging

https://kazlauskas.me/entries/errors

## Enum style

Design for use vs origin
For use
    think about recovery
    enum per match
For origin
    enum for logical class of errors
        overlap of function errors
    not one per module
    examples

Describing/categorising errors
Beware of nesting, and thus From impls
    avoid cycles - normalisation
    alternative: explicit conversion with map_err
    by extension `source` is often useless in practice and it's better to use context

## ErrorKind style

https://github.com/Azure/azure-sdk-for-rust/pull/625
https://doc.rust-lang.org/nightly/std/io/struct.Error.html

## Dynamic style
