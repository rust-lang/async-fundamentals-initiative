# With clauses

## Status

Crazy new idea that solves all problems

## Summary

* Introduce a `with(x: T)` clause that can appear wherever where clauses can appear
    * These variables are in scope in any (non-const) code block that appears within those scopes.
* Introduce a new `with(x = value) { ... }` expression
    * Within this block, you can invoke fuctions that have a `with(x: T)` clause (presuming that `value` is of type `T`); you can also invoke code which calls such functions transitively.
    * The values are propagated from the `with` block to the functions you invoke.

## More detailed look

### Simple example

Suppose that we have a generic visitor interface in a crate `visit`:

```rust
trait Visitor {
    fn visit(&self);
}

impl<V> Visitor for Vec<V>
where
    V: Visitor,
{
    fn visit(&self) {
        for e in self {
            e.visit();
        }
    }
}
```

I would like to use this interface in my crate. But I was hoping to increment a counter each time one of my types is visited. Unfortunately, the `Visitor` trait doesn't offer any way to thread access to this counter into the impl. With `with`, though, that's no problem.

```rust
struct Context { counter: usize }

struct MyNode {

}

impl Visitor for MyNode 
with(cx: &mut Context)
{
    fn visit(&self) {
        cx.counter += 1;
    }
}
```

Now I can use this visitor trait as normal:

```rust
fn process_item() {
    let cx = Context { counter: 0 };
    let v = vec![MyNode, MyNode, MyNode];
    with(cx: &mut cx) {
        v.visit();
    }
    assert_eq!(cx.counter, 3);
}
```

## How it works

We extend the environment with a `with(name: Type)` clause. When we typecheck a `with(name: value) { ... }` statement, we enter those clauses into the environment. When we check impls that contain `with` clauses, they match against those clauses like any other where clause.

After matching an impl, we are left with a "residual" of implicit parameters. When we monomorphize a function applied to some particular types, we will check the where clauses declared on the function against those types and collect the residual parameters. These are added to the function and supplied by the caller (which must have them in scope).

### Things to overcome

Dyn value construction: we need some way to permit impls that use `with` to be made into `dyn` values. This is very hard, maybe impossible. The problem is that we don't
want to "capture" the `with` values into the `dyn` -- so what do we do if somebody 
packages up a `Box<dyn>` and puts it somewhere?

We could require that context values implement `Default` but .. that stinks. =)

We could panic. That kind of stinks too!

We could limit to traits that are not dyn safe, particularly if there was a manual impl of dyn safety. The key problem is that, today, for a dyn safe trait, one can make a `dyn` trait without knowing the source type:

```rust
fn foo<T: Visitor + 'static>(v: T) {
    let x: Box<dyn Visitor> = Box::new(v);
}
```

But, now, what happens if the `Box<dyn Visitor>` is allowed to escape the `with` scope, and the methods are invoked?

Conceivably we could leverage lifetimes to prevent this, but I'm not *exactly* sure how. It would imply a kind of "lifetime view" on the type `T` that ensures it is not considered to outlive the `with` scope. That doesn't feel right. What we *really* want to do is to put a lifetime bound of sorts on the... use of the where clause.

We could also rework this in an edition, so that this capability is made more explicit. Then only traits and impls in the new edition would be able to use `with` clauses. This would harm edition interop to some extent, we'd have to work that out too.

