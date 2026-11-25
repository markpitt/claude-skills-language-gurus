# Ownership and API Design

Design APIs around borrowing and clear ownership transfer.

## Ownership First Principles

- Borrow by default (`&T`, `&str`, `&[T]`) when ownership transfer is unnecessary.
- Move ownership only when lifecycle transfer or mutation semantics require it.
- Avoid clone-heavy designs that mask unclear data flow.

## Public API Preferences

- Accept `&str` instead of `&String`.
- Accept slices (`&[T]`) instead of `&Vec<T>`.
- Return iterators or lightweight views when possible for large data paths.
- Use newtypes to encode domain constraints and prevent primitive misuse.

## Traits and Generics

- Use trait bounds that express requirements minimally.
- Prefer static dispatch unless runtime polymorphism is required.
- Keep generic signatures readable; avoid over-generalization in public APIs.

## Lifetimes

- Rely on elision and inference where clear.
- Make lifetimes explicit when needed to communicate borrowing relationships.
- Refactor ownership boundaries before introducing complex lifetime gymnastics.
