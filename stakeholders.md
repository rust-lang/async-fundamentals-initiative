# Stakeholders

## The pitch

The [async fundamentals initiative](https://rust-lang.github.io/async-fundamentals-initiative/) is developing designs to bring async Rust "on par" with synchronous Rust in terms of its core capabilities:

* async functions in traits (our initial focus)
* async drop (coming later)
* async closures (coming later)

We need feedback from people using Rust in production to ensure that our designs will meet their needs, but also to help us get some sense of how easy they will be to understand. One of the challenges with async Rust is that it has a lot of possible variations, and getting a sense for what kinds of capabilities are most important will help us to bias the designs.

**We also want people to commit to experimenting with these designs while they are on nightly!** This doesn't mean that you have to ship production software based on the nightly compiler. But it does mean that you agree to, perhaps, port your code over to use the nightly compiler on a branch and tell us how it goes. Or experiment with the nightly compiler on other codebases you are working on.

## Expected time commitment

* One 90 minute meeting per month + written feedback
    * Stucture:
        * ~30 minute presentation covering the latest thoughts
        * ~60 minutes open discussion with Tyler, Niko, other stakeholders
    * Written feedback:
        * answer some simple questions, provide overall perspective
        * expected time: ~30 minutes or less
* Once features become available (likely early next year), creating branch that uses them
    * We will do our best to make this easy
    * For example, I expect us to offer an alternative to `async-trait` procedural macro that generates code that requires the nightly compiler
    * But this will still take some time! How much depends a bit on you.
    * Let's guess-timate 2-3 hours per month

## Benefits

* Shape the design of async fn in traits
* Help ensure that it works for you
* A t-shirt!

## Number of stakeholders

We are looking for 3-5 covering various points in the design space, e.g.

* Web services author
* Embedded Rust
* Web framework author
* Web framework consumer
* High-performance computing
* Operating systems

If you have thoughts or suggestions for good stakeholders, we'd be keen to hear them.
