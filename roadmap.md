# Async fn fundamentals

This initiative is part of the overall [async vision roadmap](https://rust-lang.github.io/wg-async-foundations/vision/roadmap.html).

## Impact

* Able to write `async fn` in traits and trait impls
    * Able to easily declare that `T: Trait + Send` where "every async fn in `Trait` returns a `Send` future"
    * Traits that use `async fn` can still be [dyn safe](./design-discussions/dyn_async_trait.md) though some tuning may be required
    * Async functions in traits desugar to [impl Trait in traits]
* Able to write ["async fn drop"][async drop] to declare that the destructor may await
* Support for [async closures]

## Milestones

| Milestone | State | Key participants |
| --- | --- | --- |
| Author [evaluation doc] for [static async trait] | done 🎉 | [tmandry]
| Author [evaluation doc] for [dyn async trait]  | [done 🎉](https://rust-lang.github.io/async-fundamentals-initiative/evaluation/challenges/dyn_traits.html) | [tmandry]
| Author [evaluation doc] for [async drop] | 💤 |
| Author [evaluation doc] for [impl Trait in traits]  | done | [tmandry]
| [Stabilize] [type alias impl trait] | in-progress | [oli-obk]
| [Stabilize] [generic associated types] | [done 🎉](https://github.com/rust-lang/rust/pull/96709) | [jackh726]
| Author RFC for async fn in traits  | [done 🎉](https://github.com/rust-lang/rfcs/pull/3185) |
| Author [evaluation doc] for [async closures]  | [blog](https://smallcultfollowing.com/babysteps/blog/2023/03/29/thoughts-on-async-closures/) [posts](https://smallcultfollowing.com/babysteps/blog/2023/05/09/giving-lending-and-async-closures/) authored, doc pending | [nikomatsakis] |
| Author RFC for impl trait in traits  | [done](https://github.com/rust-lang/rfcs/pull/3425) |
| [Feature complete] for async fn in traits | done 🎉 | [compiler-errors] |
| [Feature complete] for [impl Trait in traits] | done 🎉 | [compiler-errors] |
| [Feature complete] for [async drop] | 💤 |
| [Feature complete] for [async closures] | 💤 |
| [Stabilize] async fn in traits | [proposed](https://github.com/rust-lang/rust/pull/115822) | [compiler-errors] |
| [Stabilize] impl trait in traits | [proposed](https://github.com/rust-lang/rust/pull/115822) | [compiler-errors] |
| [Stabilize] async drop | 💤 |
| [Stabilize] async closures | 💤 |

## Design discussions

This directory hosts notes on important design discussions along with their resolutions.
In the table of contents, you will find the overall status:

* ✅ -- **Settled!** Input only needed if you have identified a fresh consideration that is not covered by the write-up.
* 💬 -- **Under active discussion.** Check the write-up, which may contain a list of questions or places where feedback is desired.
* 💤 -- **Paused.** Not under active discussion, but we may be updating the write-up from time to time with details.

[nikomatsakis]: https://github.com/nikomatsakis/
[oli-obk]: https://github.com/oli-obk/
[jackh726]: https://github.com/jackh726/
[tmandry]: https://github.com/tmandry/
[compiler-errors]: https://github.com/compiler-errors/
[async drop]: ./design-discussions/async_drop.md
[async closures]: ./design-discussions/async_closures.md
[impl Trait in traits]: ./design-discussions/impl_trait_in_traits.md
[type alias impl trait]: https://github.com/rust-lang/rust/issues/63063
[generic associated types]: https://github.com/rust-lang/generic-associated-types-initiative
[static async trait]: ./design-discussions/static_async_trait.md
[dyn async trait]: ./design-discussions/dyn_async_trait.md

[evaluation doc]: https://rust-lang.github.io/wg-async-foundations/vision/how_to_vision/evaluations.html
[stabilize]: https://lang-team.rust-lang.org/initiatives/process/stages/stabilized.html
[feature complete]: https://lang-team.rust-lang.org/initiatives/process/stages/feature_complete.html
