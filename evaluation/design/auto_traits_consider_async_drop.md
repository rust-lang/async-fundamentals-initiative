# Auto traits consider async drop

One way to solve the [bounding async drop] challenge is to require that, if a type `X` implements `AsyncDrop`, then `X: Send` only if the type of its async drop future is also `Send`. The drop trait is already integrated quite deeply into the language, so adding a rule like this would not be particularly challenging.

[bounding async drop]: ../challenges/bounding_async_drop.md