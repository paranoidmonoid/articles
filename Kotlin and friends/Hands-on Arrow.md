# Hands-on Arrow

Read-only version of a talk with the same name: [slides](https://speakerdeck.com/paranoidmonoid/hands-on-arrow), [video](https://www.youtube.com/watch?v=tkl9EaUMfm8)

If you always wanted to try some fancy functional programming and/or [Arrow](https://arrow-kt.io/), then it's a right place to start.

Disclaimer: this article is intro-level and covers `NonEmptyList`, `Either` and basics of Monad comprehensions. Some definitions may be simplified for a better understanding.

### What is Arrow?

Arrow is actually not a single library but four: 

* Core (more functional programming)
* FX (functional effects and interactions with advanced stuff like `Monads`)
* Optics (for traversing and transforming immutable data structures)
* Meta (writing compiler plugins)

Sounds both exciting and scary? Don't worry, we are covering only `Core` (and a bit of `FX`) today. :)

Let's start with something simple...

### NonEmptyList

Imagine, you want to work this structure:

|Name        |Type                                   |           |Is required?|
|------------|---------------------------------------|-----------|------------|
|Payments    |List of Items(Phone, Reference, Amount)|           |            |
|Phone       |...                                    |String(100)|Required    |
|Reference   |...                                    |String(100)|Required    |
|Amount      |...                                    |String(100)|Required    |

But does the following JSON
```
{
  "payments" : []
}
```
make sense?

From the development standpoint sure, why not? An empty list is a valid List<T>. But in real life it's more complicated. To confirm payments without any payment sounds wrong.

Let's restrict it on the type level:

```kotlin
NonEmptyListOf() // doesn't compile

NonEmptyList(1, 2, 3, 4, 5) // NonEmptyList<Int>
```
To add a smooth integration with Jackson use [Arrow Integrations Jackson Module](https://mvnrepository.com/artifact/io.arrow-kt/arrow-integrations-jackson-module)

Now our domain-specific logic is reflected in our code base. Sounds easy, isn't it?

Time to move to a bit more scary this called `Either`

### Either

But let's take a step back to Java

##### Java Exceptions

Take a look on typical usage of an exception in Java:

```java
void testFun() throws SomeException {
  if (somethingWrong) throw new SomeException
}  
```
We just mark a function with `throws` and deal with it later for example like this:

```java
try {
  testFun()
} catch (SomeException e) {
  // handle
}
```

There's nothing wrong with `try-catch` it's just not always compatible with some APIs/styles. For example, the Stream API.

We simply cannot write something like this:
```java
IntStream.range(0, 10).forEach(i -> testFun())
```
Because as we defined above `testFun()` the function is throwing a *checked* exception. And it has to be checked e.g. a common approach is to delegate the exception handling to somewhere else via *unchecked* exception:

```java
IntStream.range(0, 10).forEach(hideException(i -> testFun()));
```

And then unfold it back:

```java
unhideSomeException(() -> 
  IntStream.range(0, 10)
    .forEach(
      hideException(i -> 
        testFun()
      )
    )
);

```

Where "unhide" function should have some matcher inside to "retrieve" the hidden exception. If you can. The checked part of `testFun()` would be overwritten by `hideException()` and it depends on you how would you handle that.

Now the Stream API sounds not so fun. But aren't we using Kotlin?

##### Kotlin Exceptions

In Kotlin, as in other popular JVM languages (e.g. Groovy, Scala), only unchecked exceptions are thrown during the runtime. So, we are not "forced" to have such checks.

Now the example above would be just

```kotlin
(0 until 10).forEach { testFun() }
```

(It's better to use `repeat` though)

But the thing is, we still want to handle error-flows somehow. What do we have in Kotlin for that?

##### Nulls

Null is easy to use and also Kotlin standard library has a lot of built-in functions to use, like `fun String.toIntOrNull(): Int?` which returns a nullable type. The downsides are, it's not so expressive (Why is `null` here? What exactly went wrong? Because of that, debugging may be complicated) and explicit null-checks are needed.

##### Result Type

Among built-in options Kotlin has a result type:

```kotlin
inline class Result<out T> : Serializable
```

No info about errors on compile time but at least we can do chains.

Compare the try-catch flow here:

```kotlin
fun deserialize(): Something {
   return try {
     doSomething()
   } catch (_: Exception) {
     try {
       doSomethingElse()
     } catch(_: Exception) {
       doSomethingElseAgain()
  }
}
```

And the chained one:

```kotlin
fun deserialize(): Something {
   return runCatching {
       doSomething()
   }.recoverCatching {
       doSomethingElse()
   }.getOrElse {
       doSomethingElseAgain()
   }
}
```

(Just be mindful about what exactly you can catch)

##### Sealed Type

If you can list every single outcome it makes sense to try to describe a `sealed` class. Imagine, you want to trade some stocks. You will either sell them successfully or something may go wrong due to full moon (or whatever belief of misfortune there's in your culture) or Elon Musk crashed the market by tweeting something:

```kotlin
sealed class Result

class Success : Result()

class FullMoonFailure : Result()
class ElonMuskTweetedFailure : Result()
```

##### Either

As you can see in previous example, there's only one condition for "success" but multiple reasons to "fail". And that's pretty typical everywhere. The idea of grouping by "single success" and "failure(s)" is the main concept of the `Either` class:

```kotlin
sealed class Either<out A, out B>
```

It has two subclasses called `Left` and `Right`. The first one is used to store data in case something went wrong and the second one is used in case of success. 
Just like this:

```kotlin
> Either.Left("Something went wrong")
> Either.Right(5)
```

##### Validated

`Validated` looks almost the same as `Either` class: `sealed class Validated<out A, out B>` but it's used more to describe cases when you would like to collect the result of all validations and not the first one that failed. Take an example:

```
User(phone,  name,        age)
User(“test”, XÆA-12 Musk, -10)
```

Conceptually like this:

* Either: invalid phone
* Validated: invalid phone, name, age

(You should never validate names though (classical read about [Falsehoods Programmers Believe About Names](https://www.kalzumeus.com/2010/06/17/falsehoods-programmers-believe-about-names/)))


### Monad comprehensions

Again, let's take a step back and take a look on the standard library.

##### Required/Contracts

A contract is a way to tell the compiler about our expectations. As usual, we don't like nulls and let's say we don't like to work with them:

```kotlin
requireNotNull(request.userId)
requireNotNull(request.transactionId)
```

Inside, this can look like this:

```kotlin
public inline fun <T : Any> requireNotNull(value: T?): T {
    contract {
        returns() implies (value != null)
    }
    return requireNotNull(value) { "Required value was null" }
}
```
[Source](https://github.com/JetBrains/kotlin/blob/master/libraries/stdlib/src/kotlin/util/Preconditions.kt)

If `value` was indeed `null` `IllegalArgumentException` will be thrown.

But as we discussed before sometimes we don't really want `Exceptions`. We could check and handle each case step-by-step and build our own exception response.

```kotlin
if(request.userId == null) {
  return buildExceptionResponse("userId is null")
}
if(request.transaction.id == null) {
  return buildExceptionResponse("transaction id is null")
}
// and so on for any further parameters
```
Great, now we got some repetitive patterns. Is there anything that may help us?

##### Monads
Now, back to the topic. On a basic level, you may imagine that a monad is a class with `map()` and `flatMap()` methods. 
(It's actually a bit more complicated than that, but it's outside of the article's scope)

*See docs: [map](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.collections/map.html), [flatmap](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.collections/flat-map.html)*

And that's it.

I'm pretty sure you already know some of them!
Some examples are:
* `List`
* `Either`

Let's rewrite our contract in that way so it returns Either:

```kotlin
private fun <T : Any> checkForNull(param: T?, paramName: String): Either<ErrorResponse, T> { // mapping "left" and "right" results }
```

And then we can chain multiple checks using `flatMap`:
```kotlin
val response = checkForNull(response)
    .flatMap { userId ->
        checkForNull(userId).flatMap { paymentId ->
            checkForNull(paymentId).flatMap { items ->
                checkForNull(items).flatMap { status ->
                    checkForNull(status).map {
                        ( … )
                    }
                }
            }
        }
    }
```

And now magic begins! Since it adds more and more indentation, let's rewrite it to be more readable:
```kotlin
val response = Either.fx {
    val userId = checkForNull(request.userId).bind()
    val paymentId = checkForNull(request.transaction.id).bind()
    val items = checkForNull(request.transaction.items).bind()
    val status = client.checkPaymentStatus(userId, …)
}
return response.getOrHandle { it }
```

What `bind()` does is return the expression on the right side or stops the execution returning the left value and `fx` creates the context to use the `bind()` function.

Now you and your code are amazing~~~

### Afterword

That's it for today. I hope this gives you an idea of what you can adapt in your project (at least the `NonEmptyList` shouldn't be scary!).

What would be your next steps?
* Learn about other monads in Arrow like `Option`, `IO` or `Ior`
* `Reader`, `Writer`, `State`
* ???
* Get a PhD
* ???
* `Free`, `Coyoneda`, `Cokleisli`

---

Written by [Karin-Aleksandra Monoid](https://twitter.com/paranoidmonoid) 
