# Implicit await with async drop

Consider this code:

```rust
async fn foo(input: &QueryInput) -> anyhow::Result<()> {
    let db = DatabaseHandle::connect().await;
    let query = assemble_query(&input)?;
    let results = db.perform_query(query).await;
    while let Some(result) = results.next().await? {
        ...
    }
}
```

Now let us assume that `DatabaseHandle` implements `AsyncDrop` to close the connection. There are numerous points here where `db` could be dropped (e.g., each use of `?`). At each of those points, there is effectively an implicit `await` similar to `AsyncDrop::async_drop(db).await`. It seems clear that users should not be required to manually write those things, but it is also a weakening of the existing `.await` contract (that all blocking points are visible).

