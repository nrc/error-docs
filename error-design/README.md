# Error design

How should you design error handling in your program? This half of the book aims to help with that question. There isn't one definitive answer for all programs, different programs have different requirements and there are different approaches which are popular in the community with no de facto or de jure favourite. However, there are some best practices and guiding principles that most Rust programmers agree on. 

The first chapter in this section will help you categorise and characterise errors. The second chapter will help you understand the error handling options at a high level (issues like whether to recover from an error, when to use different kinds of errors, etc.). The third chapter is specifically about designing the error types and data structures in your program.

In each chapter, but especially the third, I'll aim to give you an unbiased survey of the different approaches as well as some of my my own opinions which do not have consensus support across the Rust community. I'll try to be clear about which is which!

NOTE: this half of the book needs more editing, updating, and work in general. Please don't take it too seriously just yet.

## Section contents

### Thinking about errors

[thinking-about-errors.md](thinking-about-errors.md)

### Error handling

[error-handling.md](error-handling.md)

### Error type design

[error-type-design.md](error-type-design.md)
