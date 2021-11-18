# 📚 Explainer

> The "explainer" is "end-user readable" documentation that explains how to use the feature being deveoped by this initiative.
> If you want to experiment with the feature, you've come to the right place.
> Until the feature enters "feature complete" form, the explainer should be considered a work-in-progress.

Async functions in traits are expected to "roll out" in stages, so we actually have several explainers, one for each phase:

* [Phase 1](./explainer/phase_1.md): Stable compiler supports async fn in traits, dynamic dispatch supported via [`dyner`] crate
    * [Narrative form]()
* Phase 2: Build on async fn in traits support:
    * async closures
    * async drop
    * `AsyncRead`, `AsyncWrite`, `Stream` (the [portability initiative] is driving the design of these traits)
* Phase 3: Circle back to dynamic dispatch, incorporating the lessons we've learned from the `dyner` crate

[`dyner`]: https://github.com/nikomatsakis/dyner