# Chasing the bug: How Kotlin type system betrays us

In our of our projects we are using Kotlin spiced with [Arrow](https://arrow-kt.io/). 
More specific, let's take a look on the class called `Either`.
If you've listened to my talk about hands-on Arrow ([slides](https://speakerdeck.com/paranoidmonoid/hands-on-arrow), [video](https://www.youtube.com/watch?v=tkl9EaUMfm8)), you'll probably remember what `Either` is. But if you don't, let me remind you.
Either is a class as follows:

```kotlin
sealed class Either<out A, out B>
```

It has two subclasses called `Left` and `Right`. The first one is used to store data in case something went wrong and the second one is used in case of success. 
Just like this:

```kotlin
> Either.Left("Something went wrong")
> Either.Right(5)
```

The mnemonic rule here is "right is _right_".

Let's take a look at the following code snippet (simplified for readability):

```kotlin
when(val e: Either<A, B> = f(args)) {
    is Left -> ...
    is Right -> ...
}
```

Note that the return type of ```f(args)``` is non-nullable ```Either```.

But sometimes a test case which called this code block failed with ```kotlin.NoWhenBranchMatchedException```.

### Setup
Speaking of "sometimes", to be more precise, it happened once or twice a week. Our team is relatively small, so, let's say once per 150 builds.

This test was touching the code base with coroutines, the tests were run in parallel and as a testing framework we used [Mockito-Kotlin](https://github.com/mockito/mockito-kotlin).

Eventually, it was time for dirty hacks. No one talks about it, but we all do this. Since that test failed just once or twice a week randomly, let's wrap our test with `while(true)` loop until it eventually fails and put some good old `logger.error` or simply `println`s. ¯\\_(ツ)_/¯ 

The test failed after about 10 seconds. And guess what, the result of our function `f(args)` was... null.

### Test frameworks issues

The call of the function `f(args)` is actually mocked using Mockito-Kotlin. But sometimes, once in a week or so, there's no stubbing found for a given set of arguments. Which is why mocked `f(args)` returns `null` instead.

NB: it's still an open question, why Mockito is silent about mock not being present for those arguments. Bumping versions and forcing the test to finish don't seem to help, but whatever.

But what makes the set of arguments inconsistent then?

### Human errors

The object used for the tests looks like this: 

```kotlin
Event(
    createdAt=... //timestamp
    now=...//timestamp
    ...
)
```

Later the author of the test took `created` from expected (which was used in the mock) and `now` from actual (which was *passed* to the mock).
While most of the time those are equal when rounded (both up) to milliseconds, if `created` happens right to the current ms' end `now` will happen in the next ms-window.

![](../img/eithervsnull/milliseconds-same.svg)

Both will be rounded to `n+1`

![](../img/eithervsnull/milliseconds-different.svg)

`createdAt` would be `n+1`, but `now` happened at `n+2` when rounded up

As the test was fast enough, the latter scenario was indeed quite rare. And since types are checked on compile-time and trusted later we couldn't really expect that.

So, the following happened:

- expected timestamp in `actual` and `expected` were taken from different fields (`createdAt` and `now`)
- if `now` happened in another ms interval, that means when rounded `createdAt` != `now` and `expected` != `actual`
- Mockito-Kotlin couldn't find a stub for the current set of arguments (we had only `f(expected)`)
- no stub => use `null` as default value
- as types are checked on compile-time and not on runtime, `val e: Either<A, B> = null` was written
- null was not matched as `Left` or `Right` leading to `NoWhenBranchMatchedException`

As it was just a typo, simply fixing the source of timestamp worked and it worked like charm. But could this situation and long debug process be prevented?

### Conclusions
After this investigation I decided to have less Java in our Kotlin and migrated our project to mockk. :)

While most of the issues can be predicted on compile time, this one caught us by surpise and took a whole evening to debug. **So the less interactions with wraps instead of idiomatic libraries the better.**
