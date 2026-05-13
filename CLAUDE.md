# Error docs — Claude context

Human readers: please see the note on AI-assisted contributions in [CONTRIBUTING.md](CONTRIBUTING.md).

This is a guide (technical documentation) for errors and error handling in the Rust programming
language.

The guide is written using markdown and is built using mdbook. Use `mdbook build` to run checks and
produce an HTML rendering of the guide. mdbook and the linkcheck plugin are already installed.

The whole guide is at the draft stage and needs updating, editing, and revising. The structure of the
guide is about right (SUMMARY.md is the source of truth for structure).

## Writing style

The guide is written in American English in a semi-formal style. The tone should be like an academic
textbook, but friendly and approachable. Occasional uses of a lighter tone are fine.

The intended audience is beginner or intermediate Rust programmers (but not absolute beginners, we
can assume that they understand the language, but may not have any experience using it).

The guide targets the latest version and edition of Rust.

Do use Oxford commas, e.g., 'foo, bar, and baz', not 'foo, bar and baz'.

Capitalize and punctuate bulleted lists as if each bullet is a complete sentence.

All code blocks should have a language specified (or have an explicit `ignore`). Any Rust code
should be correct, and should be free of bugs. There might be missing type or variable declarations
or missing code marked with `// ...`, but the code which is there must not have syntax errors or
semantic errors assuming reasonable context.

No emojis.

Writing should be concise and clear. Avoid the passive voice, significance inflation, promotional
language, and other ways of making the text more verbose. 


## Review

When you review this guide look for the following:

- The writing follows the above style guidelines.
- The text is factually and technically correct.
- Code examples do not have errors.
- The text is suitable for the intended audience.
- There are no omissions about a topic.
- Check for spelling and grammatical mistakes.
- Ignore any paragraph beginning with `TODO`.
- Cross-references are linked where appropriate.
- Links are correct and appropriate.
- That the source builds correctly (usign `mdbook build`) including the link check.
