---
title: "24 Days of Hackage: async"
---

Over the last decade, it's likely that you've noticed a shift in how we use
computers. Users no longer expect to do a single task at a time, but rather
several tasks at once. This change in usage is so prevalent, users have come to
expect even single applications perform multiple steps concurrently. This change
from sequential to concurrent processing is reflected in modern consumer
processors too - its not uncommon to find dual or even quad core processors in
laptops and even phones. Given all of this, is Haskell up to the job of building
applications that meet these requirements? You bet! And with Simon Marlow's
[`async`](http://hackage.haskell.org/package/async) library, the transition is
seemless.

Fundamentally, async offers us a handful of routines that enable us to treat
concurrent work as a first class citizen. That is - a concurrent job can be
passed around in a value, its state inspected and final result extracted. To
create some asynchronous work, we simply wrap up a computation with the `async`
combinator.

For example, let's assume we have a long computation that finds a list of all
naughty children within a mile radius of a location -
`naughtyChildrenAround`. This is an expensive database query to perform, and if
we simply call it within the current thread, we'll completely block our
application. Instead, we can use the `async` operation to run this IO
computation asynchronously:

```haskell
waitExample :: IO ()
waitExample = do
  putStrLn "Requesting a list of naughty children"
  search <- async (naughtyChildrenAround "London" 1)
  putStrLn "Searching!"
```

I mentioned before that `async` gives us *first-class* asynchronous operations,
and we can see this if we take a closer look at the type of `search`. While
`naughtyChildrenAround` has type `String -> Int -> IO [Child]`, `search` does
not have type `[Child]`, but it has type `Async [Child]`. This makes it
clear that we are holding a reference to an asychronous computation, and not the
final value itself.

So what can we do with this? First, we can block the current computation to try
and extract the result:

```haskell
  searchResults <- wait search
  print searchResults
```

This is a start - we can start this operation in the background and then do some
other work. Later, we can block as late as possible. An alternative would be to
poll this asynchronous computation and ask it: are you done yet? We can do this
with `poll`. With only a little bit more work, we can now be a bit more
interactive in our program, and show an indication that we're still doing some
work and haven't just entered an infinite loop:

```haskell
  let loop = do
        maybeResults <- poll search
        case maybeResults of
          Nothing -> do
            putStrLn "Still searching..."
            threadDelay 100000
            loop
          Just r -> return r

  loop >>= print
```

However, the type of `searchResults` is a bit different this time - rather than
having a `[Child]`, we now have `Either SomeException [Child]`. This
brings us nicely onto the next feature, `async`'s exception handling.

Just running code on a different thread isn't a big deal in Haskell - `forkIO`
and lightweight threads mean we don't really even have to think about
it. However, there's a problem. When you `forkIO`, it becomes very difficult to
check whether the computation succeeded or failed. With `Async` values though,
we can easily inspect a computation to see if it failed or not.

The semantics of `wait` that we saw earlier is to rethrow any exceptions of the
asynchronous computation. This works well for some tasks, but if we use
`waitCatch`, then we can build more of a worker model, where our child threads
don't necessarily take down the parent, also giving us the option to add
logging, etc.

Finally, `async` can also be useful to distribute a large amount of work over
multiple concurrent processes. Using `mapConcurrently` we can map over any
`Traversable` concurrently, and using `race` we can race multiple computations
until one of them completes. This is really useful in very dynamic environments
where it's hard to predict what's most efficient - so instead we can just try
them all at once!

Writing anything interesting about the `async` library is a tad tricky, because
it's just so darn simple! A common mistake is to think asynchronous programming
is simple, and on the surface that seems the case. I just fork and away I go,
right? This can be fine when things operate correctly, but as things become more
complex, the intricate details become apparent. `async` is not a silver
bullet that will make all of this pain disappear. The good news, however, is
that once you've got a good idea for how you want to distribute work, `async`
deals with the nitty-gritty details of exception handling and lets you focus on
the interesting stuff.

If you want to learn more about `async`, and a bit about designing Haskell
libraries, I highly recommend
[Simon Marlow's talk on `async`](http://skillsmatter.com/podcast/home/high-performance-concurrency). If
you want to learn more about concurrent programming in Haskell, I can highly
recommend the book
[Parallel and Concurrent Programming in Haskell](http://chimera.labs.oreilly.com/books/1230000000929),
also by Simon Marlow. It's currently available for free - so that's perfect
reading for unwinding after that big Christmas dinner.
