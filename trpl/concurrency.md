% Concurrency

Concurrency and parallelism are incredibly important topics in computer
science, and are also a hot topic in industry today. Computers are gaining more
and more cores, yet many programmers aren't prepared to fully utilize them.

Rust's memory safety features also apply to its concurrency story too. Even
concurrent Rust programs must be memory safe, having no data races. Rust's type
system is up to the thread, and gives you powerful ways to reason about
concurrent code at compile time.

Before we talk about the concurrency features that come with Rust, it's important
to understand something: Rust is low-level enough that all of this is provided
by the standard library, not by the language. This means that if you don't like
some aspect of the way Rust handles concurrency, you can implement an alternative
way of doing things. [mio](https://github.com/carllerche/mio) is a real-world
example of this principle in action.

## Background: `Send` and `Sync`

Concurrency is difficult to reason about. In Rust, we have a strong, static
type system to help us reason about our code. As such, Rust gives us two traits
to help us make sense of code that can possibly be concurrent.

### `Send`

The first trait we're going to talk about is
[`Send`](../std/marker/trait.Send.html). When a type `T` implements `Send`, it indicates
to the compiler that something of this type is able to have ownership transferred
safely between threads.

This is important to enforce certain restrictions. For example, if we have a
channel connecting two threads, we would want to be able to send some data
down the channel and to the other thread. Therefore, we'd ensure that `Send` was
implemented for that type.

In the opposite way, if we were wrapping a library with FFI that isn't
threadsafe, we wouldn't want to implement `Send`, and so the compiler will help
us enforce that it can't leave the current thread.

### `Sync`

The second of these traits is called [`Sync`](../std/marker/trait.Sync.html).
When a type `T` implements `Sync`, it indicates to the compiler that something
of this type has no possibility of introducing memory unsafety when used from
multiple threads concurrently.

For example, sharing immutable data with an atomic reference count is
threadsafe. Rust provides a type like this, `Arc<T>`, and it implements `Sync`,
so it is safe to share between threads.

These two traits allow you to use the type system to make strong guarantees
about the properties of your code under concurrency. Before we demonstrate
why, we need to learn how to create a concurrent Rust program in the first
place!

## Threads

Rust's standard library provides a library for threads, which allow you to
run Rust code in parallel. Here's a basic example of using `std::thread`:

```
use std::thread;

fn main() {
    thread::spawn(|| {
        println!("Hello from a thread!");
    });
}
```

The `thread::spawn()` method accepts a closure, which is executed in a
new thread. It returns a handle to the thread, that can be used to
wait for the child thread to finish and extract its result:

```
use std::thread;

fn main() {
    let handle = thread::spawn(|| {
        "Hello from a thread!"
    });

    println!("{}", handle.join().unwrap());
}
```

Many languages have the ability to execute threads, but it's wildly unsafe.
There are entire books about how to prevent errors that occur from shared
mutable state. Rust helps out with its type system here as well, by preventing
data races at compile time. Let's talk about how you actually share things
between threads.

## Safe Shared Mutable State

Due to Rust's type system, we have a concept that sounds like a lie: "safe
shared mutable state." Many programmers agree that shared mutable state is
very, very bad.

Someone once said this:

> Shared mutable state is the root of all evil. Most languages attempt to deal
> with this problem through the 'mutable' part, but Rust deals with it by
> solving the 'shared' part.

The same [ownership system](ownership.html) that helps prevent using pointers
incorrectly also helps rule out data races, one of the worst kinds of
concurrency bugs.

As an example, here is a Rust program that would have a data race in many
languages. It will not compile:

```ignore
use std::thread;

fn main() {
    let mut data = vec![1u32, 2, 3];

    for i in 0..3 {
        thread::spawn(move || {
            data[i] += 1;
        });
    }

    thread::sleep_ms(50);
}
```

This gives us an error:

```text
8:17 error: capture of moved value: `data`
        data[i] += 1;
        ^~~~
```

In this case, we know that our code _should_ be safe, but Rust isn't sure. And
it's actually not safe: if we had a reference to `data` in each thread, and the
thread takes ownership of the reference, we have three owners! That's bad. We
can fix this by using the `Arc<T>` type, which is an atomic reference counted
pointer. The 'atomic' part means that it's safe to share across threads.

`Arc<T>` assumes one more property about its contents to ensure that it is safe
to share across threads: it assumes its contents are `Sync`. But in our
case, we want to be able to mutate the value. We need a type that can ensure
only one person at a time can mutate what's inside. For that, we can use the
`Mutex<T>` type. Here's the second version of our code. It still doesn't work,
but for a different reason:

```ignore
use std::thread;
use std::sync::Mutex;

fn main() {
    let mut data = Mutex::new(vec![1u32, 2, 3]);

    for i in 0..3 {
        let data = data.lock().unwrap();
        thread::spawn(move || {
            data[i] += 1;
        });
    }

    thread::sleep_ms(50);
}
```

Here's the error:

```text
<anon>:9:9: 9:22 error: the trait `core::marker::Send` is not implemented for the type `std::sync::mutex::MutexGuard<'_, collections::vec::Vec<u32>>` [E0277]
<anon>:11         thread::spawn(move || {
                  ^~~~~~~~~~~~~
<anon>:9:9: 9:22 note: `std::sync::mutex::MutexGuard<'_, collections::vec::Vec<u32>>` cannot be sent between threads safely
<anon>:11         thread::spawn(move || {
                  ^~~~~~~~~~~~~
```

You see, [`Mutex`](../std/sync/struct.Mutex.html) has a
[`lock`](../std/sync/struct.Mutex.html#method.lock)
method which has this signature:

```ignore
fn lock(&self) -> LockResult<MutexGuard<T>>
```

Because `Send` is not implemented for `MutexGuard<T>`, we can't transfer the
guard across thread boundaries, which gives us our error.

We can use `Arc<T>` to fix this. Here's the working version:

```
use std::sync::{Arc, Mutex};
use std::thread;

fn main() {
    let data = Arc::new(Mutex::new(vec![1u32, 2, 3]));

    for i in 0..3 {
        let data = data.clone();
        thread::spawn(move || {
            let mut data = data.lock().unwrap();
            data[i] += 1;
        });
    }

    thread::sleep_ms(50);
}
```

We now call `clone()` on our `Arc`, which increases the internal count. This
handle is then moved into the new thread. Let's examine the body of the
thread more closely:

```rust
# use std::sync::{Arc, Mutex};
# use std::thread;
# fn main() {
#     let data = Arc::new(Mutex::new(vec![1u32, 2, 3]));
#     for i in 0..3 {
#         let data = data.clone();
thread::spawn(move || {
    let mut data = data.lock().unwrap();
    data[i] += 1;
});
#     }
#     thread::sleep_ms(50);
# }
```

First, we call `lock()`, which acquires the mutex's lock. Because this may fail,
it returns an `Result<T, E>`, and because this is just an example, we `unwrap()`
it to get a reference to the data. Real code would have more robust error handling
here. We're then free to mutate it, since we have the lock.

Lastly, while the threads are running, we wait on a short timer. But
this is not ideal: we may have picked a reasonable amount of time to
wait but it's more likely we'll either be waiting longer than
necessary or not long enough, depending on just how much time the
threads actually take to finish computing when the program runs.

A more precise alternative to the timer would be to use one of the
mechanisms provided by the Rust standard library for synchronizing
threads with each other. Let's talk about one of them: channels.

## Channels

Here's a version of our code that uses channels for synchronization, rather
than waiting for a specific time:

```
use std::sync::{Arc, Mutex};
use std::thread;
use std::sync::mpsc;

fn main() {
    let data = Arc::new(Mutex::new(0u32));

    let (tx, rx) = mpsc::channel();

    for _ in 0..10 {
        let (data, tx) = (data.clone(), tx.clone());

        thread::spawn(move || {
            let mut data = data.lock().unwrap();
            *data += 1;

            tx.send(());
        });
    }

    for _ in 0..10 {
        rx.recv();
    }
}
```

We use the `mpsc::channel()` method to construct a new channel. We just `send`
a simple `()` down the channel, and then wait for ten of them to come back.

While this channel is just sending a generic signal, we can send any data that
is `Send` over the channel!

```
use std::thread;
use std::sync::mpsc;

fn main() {
    let (tx, rx) = mpsc::channel();

    for _ in 0..10 {
        let tx = tx.clone();

        thread::spawn(move || {
            let answer = 42u32;

            tx.send(answer);
        });
    }

   rx.recv().ok().expect("Could not receive answer");
}
```

A `u32` is `Send` because we can make a copy. So we create a thread, ask it to calculate
the answer, and then it `send()`s us the answer over the channel.


## Panics

A `panic!` will crash the currently executing thread. You can use Rust's
threads as a simple isolation mechanism:

```
use std::thread;

let result = thread::spawn(move || {
    panic!("oops!");
}).join();

assert!(result.is_err());
```

Our `Thread` gives us a `Result` back, which allows us to check if the thread
has panicked or not.
