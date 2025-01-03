# Rethinking the Actor Pattern in Rust with Traits

The actor pattern is a common approach to managing concurrency in applications. It promotes message-passing and avoids many pitfalls of shared, mutable state. Rust, with its ownership and concurrency model, provides multiple ways to implement actors, each leveraging the language’s powerful features differently.

This post walks through two actor implementations:

1. **A “classic” actor style**—inspired by an example from a [Rust blog](https://ryhl.io/blog/actors-with-tokio/) and video by [Alice Ryhl](https://www.youtube.com/watch?v=fTXuGRP1ee4)
2. **A more trait-based approach**—one that embraces teaching the Rust compiler how to make any type act like an “ID generator,” avoiding the need to define numerous extra structs.

Along the way, we’ll clarify the reasoning behind using traits to define behaviors and demonstrate how that can keep your code small, flexible, and composable.

A rambling video version of this content is available on my [Rust Programmer YouTube Page](https://www.youtube.com/watch?v=tOYlxaC-3QQ).

All of the source code in a devcontainer is in my [GitHub Repository](https://github.com/aughey/rust_actor_pattern).

---

## Traditional Actor Approach

Let’s start with a straightforward actor implemented by Alice. In her example, there’s a `MyActor` struct, which encapsulates:

- **A receiver** (`mpsc::Receiver<ActorMessage>`), which receives messages for the actor.  
- **An internal counter** (`next_id`), which represents the state the actor is managing.

An `enum ActorMessage` models different requests the actor can handle—in this case, just one variant that returns a unique ID. A separate handle (`MyActorHandle`) holds the sender side and exposes an async function for clients to ask for a new ID. Internally, it uses a one-shot channel to wait for the actor’s response.

Here’s an abbreviated view:

```rust
struct MyActor {
    receiver: mpsc::Receiver<ActorMessage>,
    next_id: u32,
}

enum ActorMessage {
    GetUniqueId { respond_to: oneshot::Sender<u32> },
}

impl MyActor {
    fn new(receiver: mpsc::Receiver<ActorMessage>) -> Self {
        MyActor {
            receiver,
            next_id: 0,
        }
    }

    fn handle_message(&mut self, msg: ActorMessage) {
        match msg {
            ActorMessage::GetUniqueId { respond_to } => {
            self.next_id += 1;
            let _ = respond_to.send(self.next_id);
            }
        }
    }
}
```

A companion async function, `run_my_actor`, loops forever, receiving `ActorMessage`s and passing them to `MyActor::handle_message`. The public interface for users is `MyActorHandle`, which holds a `sender` and offers `get_unique_id()`:

```rust
#[derive(Clone)]
pub struct MyActorHandle {
    sender: mpsc::Sender<ActorMessage>,
}

impl MyActorHandle {
    pub fn new() -> Self {
        let (sender, receiver) = mpsc::channel(8);
        let actor = MyActor::new(receiver);
        tokio::spawn(run_my_actor(actor));
        Self { sender }
    }

    pub async fn get_unique_id(&self) -> u32 {
        let (send, recv) = oneshot::channel();
        let msg = ActorMessage::GetUniqueId { respond_to: send };
        let _ = self.sender.send(msg).await;
        recv.await.expect("Actor task has been killed")
    }
}
```

This design effectively demonstrates the actor pattern in Rust—**it’s clear, it’s idiomatic, and it’s easy to extend** with new message types. However, the cost is a few layers of structs and enumerations, even for something as simple as returning a single ID.

---

## A Trait-Based Perspective

Now, let’s explore an alternative approach that aims to:

1. **Minimize extra structs and boilerplate**  
2. **Use traits to express a behavior** (e.g., “I can generate unique IDs”), rather than enumerating message types  
3. **Leverage Rust’s async ecosystem** without overengineering

### The `Generator` Trait

The key piece here is defining a trait—`Generator`—that states “this type can generate unique IDs asynchronously and might fail.” Here’s a minimal version:

```rust
#[allow(async_fn_in_trait)]
pub trait Generator {
    async fn get_unique_id(&mut self) -> anyhow::Result<u32>;
}
```

> **Note**: The `#[allow(async_fn_in_trait)]` attribute is needed because async functions in traits are still behind feature gates in some versions of Rust. We’re acknowledging this for the sake of demonstration.

### Teaching Basic Types to “Behave” as Generators

With this trait, we can “teach” **any** type how to provide an ID. For example:

- **`u32`**: Incrementing a simple counter until it overflows.
- **`Option<u32>`**: Return the wrapped `u32`, then become `None`.
- **`Arc<Mutex<u32>>`**: Thread-safe, lock-based ID generator.

All we need to do is implement the `Generator` trait for each type:

```rust
impl Generator for u32 {
    async fn get_unique_id(&mut self) -> Result<u32> {
        *self = self
            .checked_add(1)
            .ok_or_else(|| anyhow::anyhow!("No more IDs"))?;
        Ok(*self)
    }
}

impl Generator for Option<u32> {
    async fn get_unique_id(&mut self) -> Result<u32> {
        self.take().ok_or_else(|| anyhow::anyhow!("No more IDs"))
    }
}

impl Generator for Arc<Mutex<u32>> {
    async fn get_unique_id(&mut self) -> Result<u32> {
        let mut id = self.lock().unwrap();
        *id += 1;
        Ok(*id)
    }
}
```

Each implementation has its own logic for how to produce an ID. This alone can handle multiple use cases with a single trait definition and a few lines of implementation code.

### A Minimal Actor Using a Trait

So what happens if we want an actor-based solution—where an async task holds the state and we interact with it via message-passing—without building extra structs and enums?

We can still use the same `Generator` trait and teach it to handle requests using a **Tokio MPSC channel**. Specifically, an actor can be started with a function `actor()`, which loops and increments an internal counter. On each message, it replies via a one-shot channel:

```rust
type Request = tokio::sync::oneshot::Sender<u32>;
type RequestSender = tokio::sync::mpsc::Sender<Request>;
type RequestReceiver = tokio::sync::mpsc::Receiver<Request>;

async fn actor(mut receiver: RequestReceiver) {
    let mut next_id = 0;
    while let Some(request) = receiver.recv().await {
        next_id += 1;
        let _ = request.send(next_id);
    }
}
```

Next, we make a `Generator` implementation for `RequestSender`. Calling `get_unique_id()` creates a one-shot channel, sends the sending half to the actor, and awaits the response:

```rust
impl Generator for RequestSender {
    async fn get_unique_id(&mut self) -> Result<u32> {
        let (send, recv) = tokio::sync::oneshot::channel();
        self.send(send).await?;
        Ok(recv.await?)
    }
}
```

Finally, a **factory function** gives us a tuple:

1. A future that runs the actor,
2. A sender (now a `Generator`) that can be cloned and used freely.

```rust
pub fn create_actor_id_generator() -> (
    impl Future<Output = ()> + Send + Sync,
    impl Generator + Clone,
) {
    let (sender, receiver) = tokio::sync::mpsc::channel::<Request>(8);
    (actor(receiver), sender)
}
```

The caller decides whether to `spawn` that actor future or `await` it in a dedicated task. Because the trait-based approach just needs to know that something can provide IDs, we avoid new structs or enumerations entirely. **We rely on the compiler to hold all necessary state inside that async function.**

---

## Pulling It All Together in `main`

A simple test application illustrates how each generator is used. Let’s focus on the highlights:

```rust
#[tokio::main]
async fn main() -> Result<()> {
    // 1. Use the "Alice" actor approach
    {
        let actor = alice::MyActorHandle::new();
        let id1 = actor.get_unique_id().await;
        let id2 = actor.get_unique_id().await;
        println!("Alice: id1 = {}, id2 = {}", id1, id2);
    }

    // 2. Use a simple u32 generator
    {
        let mut generator = mine::create_u32_id_generator();
        let id1 = generator.get_unique_id().await?;
        let id2 = generator.get_unique_id().await?;
        println!("Const: id1 = {}, id2 = {}", id1, id2);
    }

    // 3. Use an Option<u32> generator
    {
        let mut generator = Some(42);
        let id1 = generator.get_unique_id().await;
        let id2 = generator.get_unique_id().await;
        println!("Optional: id1 = {:?}, id2 = {:?}", id1, id2);
    }

    // 4. Use a locked Arc<Mutex<u32>> generator
    {
        let mut generator = mine::create_locked_id_generator();
        let id1 = generator.get_unique_id().await?;
        let id2 = generator.get_unique_id().await?;
        println!("Locked: id1 = {}, id2 = {}", id1, id2);
    }

    // 5. Use the trait-based actor approach
    {
        let (actor, mut generator) = mine::create_actor_id_generator();
        tokio::spawn(actor);
        let id1 = generator.get_unique_id().await?;
        let id2 = generator.get_unique_id().await?;
        println!("Actor: id1 = {}, id2 = {}", id1, id2);
    }
    Ok(())
}
```

By the end of `main`, you’ll see each approach in action, from a `u32` counter to a fully fledged actor, all conforming to the same `Generator` trait contract.

---

## Why Use a Trait-Centric Approach?

1. **Less Boilerplate**  
   Instead of having to define a new struct or enum for each new message type or pattern, we attach a trait to any type that can produce an ID. This can drastically reduce the conceptual overhead in your code—no need to mentally track the layout of multiple structs.

2. **Composability**  
   Because traits are composable, we can add more functionality or constraints without rewriting large swaths of code. For instance, we could require a type to be `Send + Sync` to be safely shared between threads, or rely on advanced features like associated types or GATs to further refine the design.

3. **Clear Behavioral Contracts**  
   A trait is essentially a **contract**: *“If you give me a mutable reference, I will return a future that resolves to a unique ID.”* Any type that can fulfill this, whether it’s a local counter, an actor’s channel handle, or an external service proxy, can slot right in.

4. **Compiler-Assisted Design**  
   We're allowing the compiler to resolve types that implement the trait to both generate machine code for that specific implementation, and to reduce the footprint of the type to the specific action described in the trait.

---

## Final Thoughts

The classic actor pattern—exemplified by Alice’s code—remains a traditional way to structure actor implementation. Sometimes you need or want explicit structs, enumerations, and carefully managed states. Other times, especially for simpler or more focused behaviors, a trait-based style can work wonders.

By *teaching* the compiler how a type should behave, we strip away superfluous structs, reduce code noise, and keep our focus on the behavior we care about—**generating IDs**—rather than on mechanical boilerplate.

Whether you choose the trait-based approach or a more structured actor pattern depends on your project’s complexity and design goals. Rust is flexible enough to let you do either—or both—while still reaping the benefits of the language’s safety guarantees.

**Happy coding** and enjoy exploring Rust’s concurrency and trait systems for cleaner, more modular designs!

---

### Further Resources

- **The Actor Model**: Learn about the theoretical underpinnings of actor-based concurrency.  
- **Tokio**: Comprehensive docs on using Rust’s async runtime, including channels (`mpsc`) and tasks (`spawn`).  
- **Traits in Rust**: Official Rust Book section on traits to get deeper insights into leveraging traits effectively.  

Feel free to experiment with the code snippets here, swap out `u32` for other numeric types, or extend the trait for other unique ID strategies. Ultimately, it’s about finding the best balance of clarity, performance, and maintainability for your application.