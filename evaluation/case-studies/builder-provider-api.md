# Async Builder + Provider API Case Study

This case study presents a common API pattern found in builders in the AWS SDK.

## Current API

Several builders in the AWS SDK follow a "async provider" model, where the builder takes an implementation of a trait returning a future to customize behavior.

```rust=
let credentials = DefaultCredentialsChain::builder()
    // Provide an `impl ProvideCredentials`
    .with_custom_credential_source(MyCredentialsProvider)
    .build()
    .await;
```

In this example, the user is able to add a custom credentials source for a `DefaultCredentialsChain`. This credentials source is allowed to do async work upon invocation by the credentials chain. The `with_custom_credential_source` builder method takes an implementation of the `ProvideCredentials` trait:

```rust=
pub trait ProvideCredentials: Send + Sync + Debug {
    fn provide_credentials(&self) -> ProvideCredentials<'_>;
}
```

The current `ProvideCredentials` trait is a bit awkward. It expects the implementor to return an instance of [`ProvideCredentials<'_>`](https://docs.rs/aws-credential-types/0.54.1/aws_credential_types/provider/future/struct.ProvideCredentials.html) struct, which acts like a boxed future that yields `Result<Credentials, CredentialsError>`:

```rust=
struct MyCredentialsProvider;
// Implementations return `ProvideCredentials<'_>`, which is basically a boxed
// `impl Future<Output = Result<Credentials, CredentialsError>>`.
impl ProvideCredentials for MyCredentialsProvider {
    fn provide_credentials(&self) -> ProvideCredentials<'_> {
        ProvideCredentials::new(async move {
            /* Make some credentials */
        })
    }
}
```

Under the hood, when the builder's `with_custom_credential_source` is called, it boxes the `impl ProvideCredentials` and stores it for use in the `DefaultCredentialsChain` that will be built.

## With AFIT

Since `ProvideCredentials` basically returns an `impl Future` already, with AFIT, `ProvideCredentials` can instead be simplified to:

```rust=
trait ProvideCredentials {
    async fn provide_credentials(&self) -> Result<Credentials, CredentialsError>;
}
```

The user can then provide an implementation of the trait without the extra step of wrapping the function body in `ProvideCredentials::new(async { ... })`.

```rust=
impl ProvideCredentials for MyCredentialsProvider {
    async fn provide_credentials(&self) -> Result<Credentials, CredentialsError> {
        let credentials =  query_something().await?;
        // do other things like validation
        Ok(credentials)
    }
}
```

And the builder invocation remains the same...
```rust=
let credentials = DefaultCredentialsChain::builder()
    // Provide an `impl ProvideCredentials`
    .with_custom_credential_source(MyCredentialsProvider)
    .build()
    .await;
```

### Dynamic Dispatch: Behind the API

To make this change to the builder, we need to take instances of the new `ProvideCredentials` trait. Without AFIDT[^1], we can't simply box the `impl ProvideCredentials` in `with_custom_credential_source` like we were doing before.

Luckily, we can use a small type erasure hack to get around the lack of AFIDT, introducing a new trait called `ProvideCredentialsDyn`  that has a blanket impl for all implementors of `ProvideCredentials`:

[^1]: "Async functions in dyn trait", allowing traits with `async fn` methods to be object safe.

```rust=
trait ProvideCredentialsDyn {
    fn provide_credentials(&self) -> Pin<Box<dyn Future<Output = Result<Credentials, CredentialsError>> + '_>>;
}

impl<T: ProvideCredentials> ProvideCredentialsDyn for T {
    fn provide_credentials(&self) -> Pin<Box<dyn Future<Output = Result<Credentials, CredentialsError>> + '_>> {
        Box::pin(<Self as ProvideCredentials>::provide_credentials(self))
    }
}
```

This new `ProvideCredentialsDyn` trait *is* object-safe, and can be boxed and stored inside the builder instead of `ProvideCredentials`:

```rust=
struct DefaultCredentialsChain {
    credentials_source: Box<dyn ProvideCredentialsDyn>,
    // ...
}

impl DefaultCredentialsChain {
    fn with_custom_credential_source(self, provider: impl ProvideCredentials) {
        // Coerce `impl ProvideCredentials` to `Box<dyn ProvideCredentialsDyn>`
        Self { provider: Box::new(credentials_source), ..self }
    }
}
```

This extra trait is an implementation detail that is not in the public-facing, API so it can be migrated away when support for AFIDT is introduced.

A full builder pattern example is implemented here: https://play.rust-lang.org/?version=nightly&mode=debug&edition=2021&gist=8daf7b2d5236e581f78d2c09310d09ac

## Send bounds

One limitation with the proposed async version of `ProvideCredentials` is the lack of a `Send` bound on the future returned by `ProvideCredentials`. This bound is enforced by the pre-AFIT version of this trait, so any futures using the builder will not be `Send` after AFIT migration.

To fix this, we could use a return type bound[^2] on the `with_custom_credential_source` builder method:

[^2]: https://smallcultfollowing.com/babysteps/blog/2023/02/13/return-type-notation-send-bounds-part-2/

```rust
impl DefaultCredentialsChain {
    fn with_custom_credential_source(
        self, 
        provider: impl ProvideCredentials<provide_credentials(): Send>
    ) {
        // Coerce `impl ProvideCredentials` to `Box<dyn ProvideCredentialsDyn>`
        Self { provider: Box::new(credentials_source), ..self }
    }
}
```

Then the `ProvideCredentialsDyn` trait could be modified to return `Pin<Box<dyn Future<Output = Result<Credentials, CredentialsError>> + Send + '_>>`:

```rust=
trait ProvideCredentialsDyn {
    fn provide_credentials(&self) -> Pin<Box<dyn Future<Output = Result<Credentials, CredentialsError>> + Send + '_>>;
}

impl<T: ProvideCredentials<provide_credentials(): Send>> ProvideCredentialsDyn for T {
    fn provide_credentials(&self) -> Pin<Box<dyn Future<Output = Result<Credentials, CredentialsError>> + Send + '_>> {
        Box::pin(<Self as ProvideCredentials>::provide_credentials(self))
    }
}
```

Alternative and equivalent to this would be something like bounding by `T: async(Send) ProvideCredentials`, which may look like:

```rust=
impl<T: async(Send) ProvideCredentials> ProvideCredentialsDyn for T {
    fn provide_credentials(&self) -> Pin<Box<dyn Future<Output = Result<Credentials, CredentialsError>> + Send + '_>> {
        Box::pin(<Self as ProvideCredentials>::provide_credentials(self))
    }
}
```

## Usages

The SDK uses this same idiom several times:
* `ProvideCredentials`: https://docs.rs/aws-credential-types/0.54.1/aws_credential_types/provider/trait.ProvideCredentials.html
* `AsyncSleep`: https://docs.rs/aws-smithy-async/0.54.3/aws_smithy_async/rt/sleep/trait.AsyncSleep.html
* `ProvideRegion`: https://docs.rs/aws-config/0.54.1/aws_config/meta/region/trait.ProvideRegion.html

## Future improvements

With AFIDT, we can drop the `ProvideCredentialsDyn` trait and just use `Box<dyn ProvideCredentials>` as is. Refactoring the API to use AFIDT is a totally internal-facing change.