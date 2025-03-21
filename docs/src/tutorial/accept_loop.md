## Writing an Accept Loop

Let's implement the scaffold of the server: a loop that binds a TCP socket to an address and starts accepting connections.


First of all, let's add required import boilerplate:

```rust
#![feature(async_await)]

use std::net::ToSocketAddrs; // 1

use async_std::{
    prelude::*, // 2
    task,       // 3
    net::TcpListener, // 4
};

type Result<T> = std::result::Result<T, Box<dyn std::error::Error + Send + Sync>>; // 5
```

1. `async_std` uses `std` types where appropriate.
    We'll need `ToSocketAddrs` to specify address to listen on.
2. `prelude` re-exports some traits required to work with futures and streams.
3. The `task` module roughly corresponds to the `std::thread` module, but tasks are much lighter weight.
   A single thread can run many tasks.
4. For the socket type, we use `TcpListener` from `async_std`, which is just like `std::net::TcpListener`, but is non-blocking and uses `async` API.
5. We will skip implementing comprehensive error handling in this example.
   To propagate the errors, we will use a boxed error trait object.
   Do you know that there's `From<&'_ str> for Box<dyn Error>` implementation in stdlib, which allows you to use strings with `?` operator?


Now we can write the server's accept loop:

```rust
async fn server(addr: impl ToSocketAddrs) -> Result<()> { // 1
    let listener = TcpListener::bind(addr).await?; // 2
    let mut incoming = listener.incoming();
    while let Some(stream) = incoming.next().await { // 3
        // TODO
    }
    Ok(())
}
```

1. We mark the `server` function as `async`, which allows us to use `.await` syntax inside.
2. `TcpListener::bind` call returns a future, which we `.await` to extract the `Result`, and then `?` to get a `TcpListener`.
   Note how `.await` and `?` work nicely together.
   This is exactly how `std::net::TcpListener` works, but with `.await` added.
   Mirroring API of `std` is an explicit design goal of `async_std`.
3. Here, we would like to iterate incoming sockets, just how one would do in `std`:

   ```rust
   let listener: std::net::TcpListener = unimplemented!();
   for stream in listener.incoming() {

   }
   ```

   Unfortunately this doesn't quite work with `async` yet, because there's no support for `async` for-loops in the language yet.
   For this reason we have to implement the loop manually, by using `while let Some(item) = iter.next().await` pattern.

Finally, let's add main:

```rust
fn main() -> Result<()> {
    let fut = server("127.0.0.1:8080");
    task::block_on(fut)
}
```

The crucial thing to realise that is in Rust, unlike other languages, calling an async function does **not** run any code.
Async functions only construct futures, which are inert state machines.
To start stepping through the future state-machine in an async function, you should use `.await`.
In a non-async function, a way to execute a future is to handle it to the executor.
In this case, we use `task::block_on` to execute a future on the current thread and block until it's done.
