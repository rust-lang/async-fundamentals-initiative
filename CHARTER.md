# 📜 async fn fundamentals Charter
<!--
 Provide an introduction summarising the goals and motivation behind your
 initiative.
-->

This initiative is part of the wg-async-foundations [vision process].

* Able to write `async fn` in traits and trait impls
    * Able to easily declare that `T: Trait + Send` where "every async fn in `Trait` returns a `Send` future"
    * Traits that use `async fn` can still be [dyn safe][dyn async trait] though some tuning may be required
    * Async functions in traits desugar to [impl Trait in traits]
* Able to write ["async fn drop"][async drop] to declare that the destructor may await
* Support for [async closures]

## Proposal

TODO

<!--
Copy and paste the proposal into here.

Feel free to move some elements, like design questions that came up,
into the approriate section.
-->

## Membership

| Role | Github |
| ---  | --- |
| [Owner] | [tmandry](https://github.com/tmandry) |
| [Liaison] | [nikomatsakis](https://github.com/nikomatsakis) |

[Owner]: https://lang-team.rust-lang.org/initiatives/process/roles/owner.html
[Liaison]: https://lang-team.rust-lang.org/initiatives/process/roles/liaison.html

[async drop]: ./design-discussions/async_drop.md
[async closures]: ./design-discussions/async_closures.md
[dyn async trait]: ./design-discussions/dyn_async_trait.md
[impl Trait in traits]: ./design-discussions/impl_trait_in_traits.md
[vision process]: https://rust-lang.github.io/wg-async-foundations/vision.html
