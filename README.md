# async fn fundamentals initiative

![initiative status: active](https://img.shields.io/badge/status-active-brightgreen.svg)

## What is this?

This page tracks the work of the async fn fundamentals [initiative]! To learn more about what we are trying to do, and to find out the people who are doing it, take a look at the [charter].

[charter]: ./CHARTER.md
[initiative]: https://lang-team.rust-lang.org/initiatives.html

## Current status

The following table lists of the stages of an initiative, along with links to the artifacts that will be produced during that stage.

| Subproject                    | Issue    | Progress       | State | [Stage]        |
|-------------------------------|----------|----------------|-------|----------------|
| async fn                      | [#50547] | ▰▰▰▰▰▰   | ✅    | [Stabilized]   |
| static async fn in trait      | –        | ▰▱▱▱▱▱   | 🦀    | [Proposal]     |
| dyn async fn in trait         | –        | ▰▱▱▱▱▱   | 🦀    | [Proposal]     |
| async drop                    | –        | ▰▱▱▱▱▱   | 🦀    | [Proposal]     |
| async closures                | –        | ▰▱▱▱▱▱   | 💤    | [Proposal]     |

[#50547]: https://github.com/rust-lang/rust/issues/50547

<!-- TODO: Fill these in
[Proposal issue]: (https://github.com/rust-lang/lang-team/)
[Tracking issue]: https://github.com/rust-lang/rust/
-->

[Stage]: https://lang-team.rust-lang.org/initiatives/process/stages.html
[Proposal]: https://lang-team.rust-lang.org/initiatives/process/stages/proposal.html
[Experimental]: https://lang-team.rust-lang.org/initiatives/process/stages/proposal.html
[Development]: https://lang-team.rust-lang.org/initiatives/process/stages/development.html
[Feature complete]: https://lang-team.rust-lang.org/initiatives/process/stages/feature-complete.html
[Stabilized]: https://lang-team.rust-lang.org/initiatives/process/stages/stabilized.html

Key:

* ✅ – phase complete
* 🦀 – phase in progress
* 💤 – phase not started yet

## How Can I Get Involved?

* Check for 'help wanted' issues on this repository!
* If you would like to help with development, please contact the [owner](./charter.md#membership) to find out if there are things that need doing.
* If you would like to help with the design, check the list of active [design questions](./design-questions/README.md) first.
* If you have questions about the design, you can file an issue, but be sure to check the [FAQ](./FAQ.md) or the [design-questions](./design-questions/README.md) first to see if there is already something that covers your topic.
* If you are using the feature and would like to provide feedback about your experiences, please [open a "experience report" issue].
* If you are using the feature and would like to report a bug, please open a regular issue.

We also participate on [Zulip][chat-link], feel free to introduce yourself over there and ask us any questions you have.

[open issues]: /issues
[chat-link]: https://rust-lang.zulipchat.com/#narrow/stream/187312-wg-async-foundations
<!-- Should there be a dedicated team? -->
[team-toml]: https://github.com/rust-lang/team/blob/master/teams/wg-async-foundations.toml

## Building Documentation
This repository is also an mdbook project. You can view and build it using the
following command.

```
mdbook serve
```
