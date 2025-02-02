---
title: "Java's CompleteableFuture exception handling: whenComplete vs. handle"
date: 2017-10-20T23:00:15-04:00
draft: false
---

Today I learned about the behavior of thrown exceptions within asynchronous code using Java's CompletableFutures.

The question I had was:

> Is it possible to have a CompletableFuture which calls something like `.handle()`, but returns `Void`?

I was writing code that wanted to basically handle exceptional cases, but not propagate those exceptions outwards.

I had written something like:

```java
CompletableFuture.supplyAsync(() -> {
  throw new RuntimeException();
}).handle((i, err) -> {
  if (err != null) {
    throw new RuntimeException(err);
  } else {
    System.out.println(i);
    return i;
  }
}).join();
```

And since `handle` is defined as taking in a `BiFunction: (T, Throwable) -> U`:

```java
<U> CompletableFuture<U> handle(BiFunction<? super T,Throwable,? extends U> fn)
```

We're stuck with an unnecessary `return i` in our code, even though we don't care about the result of the `handle`. In this case we're just logging on success. The question was then raised:

> Why not use `whenComplete`?

Whose signature looks like:

```java
CompletableFuture<T> whenComplete(BiConsumer<? super T,? super Throwable> action)
```

And well, for one that doesn't actually solve my problem, as I still need to return something out of the `whenComplete`, but it also raised the question of what would happen in the exceptional case, and why do these two seemingly similar methods even exist?

The answer here is somewhat that they handle errors differently. If we look at the _totally clear_ JavaDocs for `handle`:

> When this stage is complete, the given function is invoked with the result (or null if none) and the exception (or null if none) of this stage as arguments, and the function's result is used to complete the returned stage.

and for `whenComplete`:

> When this stage is complete, the given action is invoked with the result (or null if none) and the exception (or null if none) of this stage as arguments. The returned stage is completed when the action returns. If the supplied action itself encounters an exception, then the returned stage exceptionally completes with this exception unless this stage also completed exceptionally.

It's totally clear what happens right? I mean from reading those two blurbs I'm sure you could easily predict the result of this snippet:

```java
CompletableFuture.supplyAsync(() -> {
  throw new RuntimeException("1");
}).whenComplete((i, err) -> {
  System.out.println("hello");
  throw new RuntimeException("2");
}).join();
```

You might assume that you'd get back `RuntimeException("2")` since thats the last thing thrown in the chain, and it seems like `whenComplete` takes in an error, so that's probably not just hanging around, right? Well, if we run the code we see:

```java
hello
Exception in thread "main" java.util.concurrent.CompletionException: java.lang.RuntimeException: 1
  at java.util.concurrent.CompletableFuture.encodeThrowable(CompletableFuture.java:273)
  at java.util.concurrent.CompletableFuture.completeThrowable(CompletableFuture.java:280)
  at java.util.concurrent.CompletableFuture$AsyncSupply.run(CompletableFuture.java:1592)
```

What even happened here to see this output? We see `hello`, so we know that the code from the `whenComplete` block executed, but that exception looks like it's from the `supplyAsync` call. So not only did our second exception simply disappear, but `whenComplete` also wrapped our inital exception in a `CompletionException`. Fantastic.

Imagine if we had wanted to handle that error somehow, and done something like:

```java
CompletableFuture.supplyAsync(() -> {
  throw new RuntimeException("1");
  return 1;
}).whenComplete((i, err) -> {
  if (err != null) {
    throw new IllegalArgumentException(err.getMessage());
  }
}).join();
```

Well again, all we're going to get back is our original exception:

```java
Exception in thread "main" java.util.concurrent.CompletionException: java.lang.RuntimeException: 1
```

So what did we learn?

1. There doesn't seem to be a great way to handle exceptions while returning `CompletableFuture<Void>`
2. If we want to actually handle any errors specially, we need to use `handle` which explicitly doesn't propagate errors for us, instead allowing us to take action on errors explicitly. This means we can either squash them, or propagate changed versions of them.
3. `CompletionException`s are basically going to spring up in our code when using `CompletableFuture`s and there's not a whole lot we can do about it, besides liberally using `handle`? We can also use `exceptionally` to deal with this problem, but we can save that for another post.
