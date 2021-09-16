# Implementing AsyncRead

`AsyncRead` is being used here as a "stand-in" for some widely used trait that appears in the standard library. The details of the trait are not important.

## Self is send

In this scenario, the Self type being used to implement is sendable, but the actual future that is created is not.

```rust
struct MySocketBuddy {
    x: u32
}

impl AsyncRead for MySocketBuddy {
    async fn read(&mut self, buf: &mut [u8]) -> io::Result<usize> {
        let ssh_key = Rc::new(vec![....]);
        do_some_stuff(ssh_key.clone());
        something_else(self.x).await;
    }
    // ERROR: `ssh_key` is live over an await;
    //        Self implements Send
    //        therefore resulting future must implement Send
}
```

## Self is not send

In this scenario, the Self type being used to implement is not sendable.

```rust
struct MySocketBuddy {
    x: u32,
    ssh_key: Rc<Vec<u8>>,
}

impl AsyncRead for MySocketBuddy {
    async fn read(&mut self, buf: &mut [u8]) -> io::Result<usize> {
        do_some_stuff(self.ssh_key.clone());
        something_else(self.x).await;
    }
    // OK
}
```