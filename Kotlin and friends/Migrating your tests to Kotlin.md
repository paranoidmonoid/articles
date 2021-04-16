# Migrating your tests to Kotlin

*This article is co-authored with [Alex Levin](https://twitter.com/Jellymath)*

Kotlin is a great language with lots of great features. If you are thinking about migrating your project to Kotlin, why
not to start with tests? In this article we will share tips to make your transition smooth.

### Tip 1: Decide, which framework you will use

Most likely you are already using mockito. In that case, start with adding the following dependencies to your project:

- [Kotlin](https://search.maven.org/artifact/org.jetbrains.kotlin/kotlin-stdlib-jdk8)
- [Kotlin Test Support for JUnit5](https://search.maven.org/artifact/org.jetbrains.kotlin/kotlin-test-junit5)
- [Mockito-Kotlin](https://search.maven.org/artifact/com.nhaarman.mockitokotlin2/mockito-kotlin)

In the future (when you switch your main codebase, too) you may benefit from switching to Kotlin-first frameworks (like
[mockk](https://github.com/mockk/mockk), [kotest](https://github.com/kotest/kotest), [strikt](https://github.com/robfletcher/strikt) and such). This goes handy if you want a more null-safe environment.

*The following article will use mockito*

### Tip 2: Use auto-converter from Java to Kotlin (no, seriously)

If you use IntelliJ Idea that option is available via Option+Shift+Command+K on Mac or just do double Shift and type
something like "Java to Kotlin".

Start with converting old simple tests. There's a good chance that everything will work out of the box.

After that we recommend replacing Java functions from JUnit and Mockito to Kotlin function from `kotlin-test-junit`
and `kotlin-mockito`.

### Tip 3: Use Kotlin

Make yourself familiar with the standard library.

##### Instead of Java's Stream API you can use Kotlin Collections, which give you nicer syntax:

Before:

```java
paymentInstruments.stream().filter(pi -> Objects.equals(publicId, pi.getId())).findFirst().orElseThrow()
```

After:

```kotlin
paymentInstruments.first { it.id == publicId }
```

Also change Java-way collection creation to Kotlin ones (
e.g. `List.of, Collections.asList, Collections.singletonList → listOf`)

##### Use whitespaces in tests' names

```
fun subServiceIsInvoked() → fun `Sub Service is invoked`()
```

Now long names are easier to read.

##### Use `lateinit var` instead of assigning `null`

Consider having a service that looks like this:

```kotlin
class Service(val subService: SubService)
```

After auto-conversion of tests you may have this code pattern:

```kotlin
@InjectMocks
val service: Service? = null

@Mock
val subService: SubService? = null
```

To avoid unnecessary null-checks, consider rewriting it using `lateinit var`:

```kotlin
@InjectMocks
lateinit var service: Service

@Mock
lateinit var subService: SubService
```

NB: Perharps you might consider avoiding mocks injection, as it may make it difficult to track if the mock is actually used or
not

### Tip 4: Enjoy reified generics

Mockito matchers are slightly different in the mockito-kotlin library. Because of reified generics in most cases you don't
need to specify type explicitly:

Before:

```java
doReturn(result).when(client).getCards(any(Data.class), any(Request.class));
```

After:

```kotlin
doReturn(result).whenever(client).getCards(any(), any())
```

Use `any()` for non-nullable matching, otherwise use `anyOrNull()`.

In some cases you still need to write types when there's ambiguity (more than one method may match).

NB: `whenever` is used instead of `when` because `when` is the keyword in Kotlin which means that its name should be escaped with backticks 
(and that's not looking very nice usually)

Another good example would be simplifying exception checking.

Before:
```java
assertThrows(SomeException.class, () -> {
    service.doSomething();
});
```

After:
```kotlin
assertFailsWith<SomeException> { service.doSomething() }
```


### Tip 5: Use scope functions and DSL based on them

##### Mock creation

You can put the mock-specific behavior to the relevant mock for extra readability.

Before:

```java
final Service service = mock(Service.class);
when(service.getCards(any())).thenReturn(response);
```

After:

```kotlin
val service: Service = mock {
    on { getCards(any()) } doReturn response
}
```

##### Assertions

Same for assertions. As a bonus, you can call fields of the scoped object without explicit object's name.
For that you can use `with` scope function:

Before:

```java
assertEquals(request.getAmount(), purchase.getAmount());
assertEquals(request.getShopConfigurationId(), purchase.getConfiguration().getId());
assertEquals(request.getWalletId(), purchase.getWallet().getPublicId();
assertEquals(request.getDescription(), purchase.getDescription());
assertEquals(request.getExternalReference(), purchase.getExternalReference());
```

After:

```kotlin
with(purchase) {
    assertEquals(request.amount, amount)
    assertEquals(request.shopConfigurationId, configuration.id)
    assertEquals(request.walletId, wallet.publicId)
    assertEquals(request.description, description)
    assertEquals(request.externalReference, externalReference)
}
```

### Wrapping up

We hope those ideas will help you to introduce Kotlin in tests. If you are not ready for rewriting old tests, 
you can also start with adding new tests only in Kotling and occasionally convert the older ones. And most importantly, enjoy!
