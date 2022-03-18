# User-facing summary

![planning rfc][]

{{#include ../badges.md}}

This is a brief summary of the user-facing changes.

* Extend the definition of `dyn Trait` to include:
    * Async functions
    * Functions that return `-> impl Trait` (note that `-> (impl Trait, impl Trait)` or other such constructions are not supported)
        * So long as `Trait` is dyn safe
* Extend the definition of `impl` to permit
    * `#[dyn(box)]` and `#[dyn(identity)]` annotations
* TBD
