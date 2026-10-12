# Overview

This book is intended for anybody who wants to get started with microcontroller programming in Rust using [Embassy](https://embassy.dev).

It is based on experience from previous work on Embassy projects and focuses on the things that matter in practice:
how the software layers fit together, which Embassy crates you will run into, how different buses shape your firmware,
and how to structure drivers and application logic so they stay testable and portable.

The book was put together in a short time frame. It gives you an overview and pointers to get going,
but it is missing some of the more in-depth information. Where that is the case, the linked documentation and examples are the best place to dig deeper.

Some familiarity with Rust (ownership, traits, `async`/`.await`) is assumed.
