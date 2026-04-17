# Cheat sheet / useful things to know:

## async runtime:

you will basically want an async runtime, tokio is pm the industry standard, there are smaller alternatives to it ie (ie smol) and others built for performance (ie compio)
for the sake of exercise i recommend you use tokio since that's what we're using internally, but the others are good to know about as well.

## useful crates:

`futures`: is pretty nifty, it has nice utilities for streams such as for_each_concurrent / stream.buffered(N), and other generally useful async related utilities

`tracing`: is nice if you want to log things, it's also an instrumentation framework

`clap`: a must have if you want to parse cli arguments, i recommend you use the derive feature, ask the llm about it, it has amazing ergonomics, you basically only have to define a struct and use some attributes over your fields (attributes are the #[] thingy, often invoking macros or talking to the compiler) 

`serde`: basically the king of serialization, a MUST have, you can just do this:

```rust
use serde::{Serialize, Deserialize};
#[derive(Serialize, Deserialize)]
struct Whatever {
// content
}

// and then your struct can be serialized to and from basically any format known to man
// through third party crates that expect the trait to be implemented for your struct.
// ie:

let t: Whatever = serde_json::from_str(&json)?;
```

`serde_json`: a json parsing crate based on `serde`

`serde_with`: adds extra helpers to serde, sometime useful

`postcard`: similar to `bincode` (which was abandoned), super useful to binary pack stuff, format is also stable, also works with `serde`

`bytes`: kinda self explaining, useful to work with bytes

`reqwest`: goto http library, there are smaller ones ie hyper, a gotcha about it is that you should avoid recreating a client instance for each request, you can use the `rustls`
feature flag to not depend on openssl if you want, also will generally make it faster to instantiate a new `Client`.

`sqlx`: nice driver for all sql related databases (sqlite, mysql, postgres), it optionaly can also do compile time checks of your query against your database, the sqlx-migrate tool that comes with it is also nifty for migration
it doesn't rely on serde but it has similar derives that allow you to directly deserialize your queries into your struct or `Vec<your_type>`

`itertools`: basically add many cool adaptors to iterators

`rayon`: probably won't be needed for this exercise but pretty nice crate to process iterators / data in parallel using multithreading

`governor`: generally useful for ratelimiting stuff

### Error handling:

for application code `anyhow` is pretty handy, basically a nicer `Result<T, Box<dyn std::error::Error>>`,
also has the `bail!()` macro which is pretty nifty.
it's generally pretty lazy when you need to do something fast without having to think about defining your error types,
it should not be used for library code and i tend to consider its use as tech debt.

the better thing to do, especialy for library code is to use `thiserror` which is basically just syntactic sugar macros for defining your error types
alternatively, you can also just not use any error crate and ship your own definition, an error type basically just need to implemment the `Error` and `Display` traits, you can find
more info about it here: https://doc.rust-lang.org/rust-by-example/error/multiple_error_types/define_error_type.html

### futures::Stream:

think of streams like async iterators for which the data is still incomming, like iterators they are lazy, ie not doing anything until consumed.
you can also write functions that take a stream as input, apply some modifications to it (ie map, filter etc) then return a stream as output.
this function doesn't do any processing as that only happens once the stream is consumed.

streams can be a nice way to concurrently do things ie:
```rust
futures::stream::iter(0..100) // converts the range into a stream
.for_each_concurrent(concurency, |n| async move {
    let result = some_async_function(n).await;
    // handle result here
})
```

you may also use buffered and buffered_unordered instead of for_each_concurrent, but you'll still need to consume the stream in some ways.
anyway, most iterator functions work on stream, ie (map, filter, filter_map, fold) you can look in the documentation for them, some are very nifty.

## borrow checker sheenanigans:

you may want to ask the llm or look up about the borrow checker and lifetimes, it's generally what people struggle the most with when they come to the language.
eventualy it'll become automatic and you won't think about it anymore but it can be confusing at first.
but they are essentialy what rust's memory safety is built upon.

the tldr is that for a value:

- you can have as many immutable references (&T) as you want
- but only one mutable reference (&mut T) at a time
- and you can't have both at the same time
- references must always be valid (no dangling pointers)

lifetimes are the borrow checker's way of tracking how long references are valid, most time they will be infered automaticaly by the compiler, but in some crates
you may need to hint it yourself.
ie:

```rust
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
    if x.len() > y.len() { x } else { y }
}
```
this basically tells the compiler that the returned reference will live at least as long as both input references.

there are 2 ways to get around the borrow checker, either use unsafe (unrecommended), or use a type allowing sharing references, ie `Arc<T>` when you need shared ownership.
and Arc<Mutex<T>> when you need shared ownership and mutability, Mutex<T> being a type with interior mutability that ensures only one thread can access the data at a time.
those type generally use unsafe internally.

you may also want to use Arc<RwLock<T>> instead of a Mutex as RwLock allows more than one reader but only one writter.
generally you can look at `std::sync::*` for those, it also includes atomics.

a good thing to know is that `tokio` has an async equivalent for many of those, a reason to use them instead of the std versions is to avoid blocking the thread.

## channels:

either in `std::sync::` or `tokio::sync::` there are many kinds of channels and i'll let you read the documentation about them, but they are super useful
for passing data around, notifying other threads of events etc.
if you need multiple producer, multiple consumer, there are crates for it, (ie `async_channel`, `flume`).
channels consumers can also be turned into `futures` (often using unfold) streams which may be a nifty way to consume them.

## unsafe:

sometime you may use unsafe, but that is very rare and also should be avoided in case it can be and getting the last bit of performance isn't a concern.
in unsafe code, most of rust's checks are still active, but it gives you those extra "superpower" which may lead to memory corruption if not careful:

- Dereference a raw pointer.
- Call an unsafe function or method.
- Access or modify a mutable static variable.
- Implement an unsafe trait.
- Access fields of unions.

you can read more about it here :
https://doc.rust-lang.org/book/ch20-01-unsafe-rust.html

generally the language treat unsafe blocks like an axiom, it makes the assumption that those blocks do not lead to memory corruption bugs
and given that assumption to be true, the rest of your application should be free of them.\
ommiting rare compiler bugs or unidiomatic code written specificaly to target those bugs.

## useful builtin derives

`Default` is nice, it allows to initialize a struct with `Foo::default()` or `let foo: Foo = Default::default()`;\
`Clone`, makes your struct clonable, for references it is cheap, for heap allocated things it may not be ie Strings, Vec.\
`Debug`, makes instances of your type printable with `println!("{:?}", foo)`

you can use them like that :
```rust
#[derive(Default, Clone, Debug)]
struct Foo {
}

```

# language quirks:

the question mark operator "?" is super useful for handling errors, you can read the doc about it:
https://doc.rust-lang.org/rust-by-example/error/result/enter_question_mark.html

## async traits:

there is some history regarding traits with async methods, you can find tons of blogposts about it and it's still an ongoing issue.
this is why the `async_trait` crate exist and you probably will want to use it, since rust 1.75 it is no longer necessary unless you need object safe traits (ie: Box<dyn Trait>).

## orphan rule

you cannot implement a foreign trait for a foreign type, this is where the newtype pattern becomes handy.
you can do `struct YourNewType(LibraryType)` and then implement the trait you want for that new type.

## documentation

documentation is built into the language, which is neat ie:

```rust
/// you can document things like that
struct Foo {
}

```
then this doc can be generated with `cargo doc`
you can read more about it here : https://doc.rust-lang.org/reference/comments.html

## clippy

the command `cargo clippy` will be useful to improve code quality, basicaly compiler suggestions on how you can improve your code even if it is valid.

## lints, allow, deny, attributes

you can hint the compiler about warning / allowing things or not.
ie:

`#![allow(non_camel_case_types)]` on top of a file to disable warnings about it.\
you can also use it without the exclamation mark over a struct so that it doesn't apply to the whole module, ie :
```rust
#[allow(non_camel_case)]
struct snake_case {
}

```

likewise you can use deny ie `#![deny(clippy::unwrap_used)]`\
lastly these can be defined in your Cargo.toml instead of rust code, look up `lints.rust` and `lints.clippy`.

the `#[must_use]` attribute is also worth mentioning

## conditional compilation

you can use the `#[cfg(...)]` attribute for conditional compilation, very often used with feature flags

## metaprogramming:

rust has basically 2 main ways to do metaprogramming, the first one is macro_rules which basically is just a pattern matcher that replaces your pattern with code.\
it is pretty limited in what it can do but still useful, more things can be done in them using `pastey` (a fork of `paste` which was archived).\
it is worth noting that they are hygienic, i'll let you search what that means.

and proc_macros, which are more annoying to set up because as of now you need a separate crate to define them, but they are also much more powerful.\
they are basically functions that take `TokenStream` (ie code) as input, and output a `TokenStream`, within those functions you can do basically anything
even http request, database request, there is not realy much limitations, except that those macros run at compile time to generate code.\
useful crates to know about are `syn` and `quote` which are nice crates for parsing and generating TokenStream that are often used by proc macros.\
in practice you'll almost always use `proc-macro2` instead of the standard proc-macro crate directly,
as it produces tokens that can be used outside of a proc-macro context, which syn and quote also depend on

there are basically 3 types of those macros:
- function like macros, called like : `foo!(a,b,c)`
- attribute macros, used to implement attributes : `#[your_macro]`
- derive macros, ie `#[derive(Serialize)]`

lastly there are crates like `crabtime` trying to bridge the gap between macro_rules that can be used anywhere but are limited and proc_macros which are more powerful but need
their own crate

you may be interested in using `cargo-expand` which expands macros and is useful for developing them.

## modules and workspaces:

### modules:
rust has a module system that allows you to organize your code, a module is declared with the `mod` keyword:
```rust
mod my_module {
    pub fn my_function() {
        // content
    }
}
```
by default everything in a module is private, you need to use `pub` to make things public.
there is also `pub(crate)` which makes something public only within the crate, and `pub(super)` which makes it public to the parent module.

modules can also live in their own file, if you declare `mod my_module;` without a body, the compiler will look for either:
- `my_module.rs`
- `my_module/mod.rs`

you can nest modules as deep as you want, and the file structure mirrors the module hierarchy.

to refer to things across modules you use the `use` keyword or the full path with `::`, ie `crate::my_module::my_function()`.
`super::` refers to the parent module and `crate::` refers to the root of the crate.

### workspaces:

when you have multiple related crates, you can group them into a workspace, it's just a `Cargo.toml` at the root that looks like this:
```toml
[workspace]
members = [
    "crates/*", // you can also use a wildcard
    "my_crate",
    "my_other_crate",
]
```
each member is its own crate with its own `Cargo.toml`, but they all share the same `Cargo.lock` and the same build output directory, which means they also share compiled dependencies, which speeds up build times a lot.

crates within a workspace can depend on each other like so:
```toml
[dependencies]
my_crate = { path = "../my_crate" }
```

you can also define shared dependencies versions at the workspace level to avoid having to repeat yourself:
```toml
[workspace.dependencies]
tokio = { version = "1", features = ["full"] }
```
and then in a member crate:
```toml
[dependencies]
tokio = { workspace = true }
// or shorter
tokio.workspace = true
```

workspaces are super handy when for instance you want to separate your library code from your binary, or when you have proc macros (remember those need their own crate 😉).

## compilation tips:

the default linker is not the fastest, i recommend you to install `mold`, and add that in `~/.cargo/config.toml`
```toml
[target.x86_64-unknown-linux-gnu]
linker="/usr/bin/clang"
rustflags = ["-C", "link-arg=--ld-path=/usr/bin/mold"]
```
this will make building quite a bit faster as a good chunk of the time is spent on linking.

also separating a project into multiple crates using a workspace will make incremental compilation faster as only crates that have been changed will need to be recompiled.\
if you want to go futher you can look into `cranelift` which is a new compilation backend that can be faster, it may be worth setting it up as the compilation backend for debug builds.

## cargo quirks

you'll want to use `edition = 2024`

`cargo test` is good to know, you can learn about how to write them yourself.

`cargo-watch`: can be useful

`cargo-script`: can be nifty if you end up liking rust so much that you want to write your scripts in it

`flamegraph`: is pretty cool for profiling if you ever need to


# Honorable mentions:

i may latter add a section on Clone and Copy, you can look them up yourself in the meanwhile.\
worth mentioning you can also Arc<dyn Trait> you don't necessarily need `Box<>`

`todo!()`, `dbg!()`, `panic!()`, `unreachable!()` and `unimplemented!()` are worth mentioning.

`#[non_exhaustive]` is good to know about

# dev workflow:

i highly recommend you use the rust-analyzer lsp and have nice editor integration, being able to read the documentation directly from the autocompletion menu is a lifechanger.
also having a shortcut to show info and documentation about a type is nice.

# Official Documentation :

rust has some pretty good official documentation:

https://doc.rust-lang.org/book/title-page.html

https://doc.rust-lang.org/rust-by-example/

for any crate you can also search "your_crate_name doc rs", and you should find the docs.rs documentation
you can also use `cargo doc` to generate the documentation of a project and all the crates it depends on, useful if you want to have offline documentation

also, this repo may be useful to you : https://github.com/rust-unofficial/awesome-rust
