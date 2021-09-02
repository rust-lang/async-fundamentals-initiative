# 📜 async fn fundamentals Charter
<!--
 Provide an introduction summarising the goals and motivation behind your
 initiative.
-->

* Able to write `async fn` in traits and trait impls
    * Able to easily declare that `T: Trait + Send` where "every async fn in `Trait` returns a `Send` future"
    * Traits that use `async fn` can still be [dyn safe](./async_fn_fundamentals/dyn_async_trait.md) though some tuning may be required
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

<!-- TODO: Fix up these links

[async drop]: ./async_fn_fundamentals/async_fn_fundamentals.md
[async closures]: ./async_fn_fundamentals/async_closures.md
[impl Trait in traits]: ./async_fn_fundamentals/impl_trait_in_traits.md
[type alias impl trait]: ./async_fn_fundamentals/tait.md
[generic associated types]: ./async_fn_fundamentals/gats.md
[static async trait]: ./async_fn_fundamentals/static_async_trait.md
[dyn async trait]: ./async_fn_fundamentals/dyn_async_trait.md

-->
