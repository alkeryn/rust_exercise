# What is this about?:
the point of this is to give an exercise to new hires so they become more familiar with rust and more specificaly async rust.\
this isn't intended for juniors but software engineers that already know the craft but may not be very familiar with rust itself yet.\
if you are still learning about programming you can try to do it but this may be overwhelming and i'd recommend you to start with simpler things.

# Exercise:

the task is basically to build an indexer for sequentialy identified elements.
ie pick any crypto, or weather station, sattelite imagery etc.
idealy that data is queried over an http api, if it's a tcp or udp api i'm also fine with it, just don't reimplement http yourself, you must use async.

now, on startup, if it's the first time running pick the last 100 blocks / id / elements.\
download them concurrently\
store them in a database\
then poll for new elements and keep in sync with your source.\
if it isn't the first run, continue from where you left off.

preferably choose a data source with frequent enough events such that you don't have to wait a day for the next one, may be easier for testing but that's your choice.\
lastly, provide an http api that allows to query the indexer for elements, i recommend you use `axum`, `actix` is not too bad either and has good documentation.\
if an element that's not yet cached / stored is queried, download and store it then send it back.

constraints:

the storage solution must be abstracted away such that it could easily be swaped (possibly at runtime ie through a config file), ie use a Trait.
elements needs to be downloaded concurrently but stored in order (you probably want to use `futures::stream::StremExt::{for_each_concurrent, buffered}` and channels)
you can use sqlite, postgres, etc whatever you want as long as your primary storage solution is used with async, for sql stuff i'd recommend the sqlx crate.
provide some logging of what's happening, you could use the tracing or log crate to do so.
even if it could technicaly be done without it, you must use async, the whole point of the exercise is to learn about it.

ie:

```rust
trait Database {
// definition here
}
// and then you'd use Box<dyn Database> or Arc<dyn Database>
// in your application code so the storage could easily be swaped
// dynamic dispatch could be avoided with a Struct<T>,
// makes sense for application where type is defined at compile time
// but here we may want to be able to pick the storage medium with a config file
// there is also some async_trait rabbit hole you may want to get into
```

## extra:

if you want do to something more at the end of the exercise, maybe write some UI for visualizing the data using `dioxus`.

# Cheat sheet

If you want to learn more about the language, i wrote some quick introduction documentation in [./documentation.md](documentation.md)
