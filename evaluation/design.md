# Design documents

This section contains detailed design documents aimed at various challenges.

| Document                         | Challenges addressed                         | Status |
| -------------------------------- | -------------------------------------------- | ------ |
| [Implied Send]                   | [Bounding futures]                           | ❌     |
| [Trait multiplication]           | [Bounding futures]                           | 🤔    |
| [Inline async fn]                | [Bounding futures], [Dyn traits] (partially) | 🤔    |
| [Custom dyn impls]               | [Dyn traits]                                 | 🤔    |
| [Auto traits consider AsyncDrop] | [Bounding drop]                              | 🤔      |

[Implied Send]: ./design/implied_send.md
[Trait multiplication]: ./design/trait_multiplication.md
[Inline async fn]: ./design/inline_async_fn.md
[Custom dyn impls]: ./design/custom_dyn_impls.md

[Bounding futures]: ./challenges/bounding_futures.md
[Dyn traits]: ./challenges/dyn_traits.md
