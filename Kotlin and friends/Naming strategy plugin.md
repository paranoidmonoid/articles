### Arrow Meta and solving the issues of Kotlin serialization library by writing a compiler plugin

I love writing in Kotlin. I also prefer to use Kotlin-first libraries to avoid
e.g. [tricky problems with nulls](https://paranoidmonoid.github.io/articles/Chasing%20the%20bug/EitherVsNull) and
other "Java"-related problems. Because of that, I wanted to migrate my project
to [kotlinx.serialization](https://github.com/Kotlin/kotlinx.serialization) instead
of [Jackson](https://github.com/FasterXML/jackson-module-kotlin).

And then I decided against it.

One of the problems is, this library doesn't have a global naming strategy.

In [Jackson](https://github.com/FasterXML/jackson-module-kotlin) you could define a global naming strategy in your
object mapper:

```kotlin
   val objectMapperUpperCamelCase = jacksonObjectMapper().configObjectMapper().copy().apply {
    propertyNamingStrategy = PropertyNamingStrategies.SNAKE_CASE
}
```

`objectMapperUpperCamelCase.writeValueAsString()` will rename all the values according to the given strategy (in this
case `SNAKE_CASE`)

This is useful when you have to integrate with other services written in various languages with different naming
conventions because I don't really want to name my variables in my Kotlin code with `snake_case` mixed
with `camelCase`. Or add to every single variable a `@SerialName`. I'm just lazy.

And like every lazy engineer I decided to spend a lot of time so I could stay lazy.

I wanted to try [Arrow Meta](https://meta.arrow-kt.io/) for a long time. It's a meta-programming library that 
"cooperates with the Kotlin compiler in all its phases" which opens a way to write compiler plugins, linters or add
custom code refactoring. Sadly, the library doesn't have a lot of docs and descriptions for the beginners, so, today we
are going to learn by example.

### Introduction

Today we are going to build a zoo: various animals (actually, only cats) from various sources (= 3rd party services)with
various standards of documentation (in JSON format with different naming policies).

A generic cat is described as usual via data class:

```kotlin
@Serializable
data class Cat(val name: String, val age: Int, val ownerName: String, val breed: String)
```

Kotlin Serialization would expect the `ownerName` field being called in incoming JSONs exactly like this, in lower camel
case. So, no `owner_name`, `owner-name` or `oWnErNaMe` styles are allowed. But today we are "integrating" only with a
cattery that uses the snake case.

My idea was, can we add a `@SerialName` annotation automatically to the fields with a proper renaming?

For the sake of simplicity, today we are hardcoding the snake case naming strategy but this concept may be customized
for anything else.

### Setting-up project

To start writing your plugin, you need to set up a multi-module project. The easiest option for now would be taking the
structure from [Arrow Meta examples](https://github.com/arrow-kt/arrow-meta-examples). Choose the one that suits you
best. Also take a look at the [docs](https://meta.arrow-kt.io/setup.html)

### Code

#### Create an extension function

First, create an extension function for Meta interface, which is a core entry point:

```kotlin
val Meta.serialNameGenerator: CliPlugin
    get() = TODO()
```

After that you need to decide on a name and call the `meta` function. In my case, is simply called it "Serial Name
Plugin"

```kotlin
val Meta.serialNameGenerator: CliPlugin
    get() =
        "Serial Name Plugin" {
            meta(TODO("add plugin logic"))
        }
```

#### Register your extension

```kotlin
class SerialNamePlugin : Meta {
    @ExperimentalContracts
    override fun intercept(ctx: CompilerContext): List<CliPlugin> =
        listOf(
            serialNameGenerator
        )
}
```

#### Decide the phase

There are multiple places you can apply your logic to like:

* `classDeclaration` - class signature (name, annotations, parameters etc.)
* `finallySection` - `finally` block
* `destructuringDeclaration` - like in `val (first, second) = listOf(first, second)`
* `classBody` - functions, properties, companions

And others

I decided to go with `classDeclaration` as it was easier for me to get all annotated classes and transform the
parameters:

```kotlin
classDeclaration(this, { isAnnotatedWith("@Serializable".toRegex()) }) { ... }
```

You can also filter your classes by name like `{ name == "Cat" }`, type with `isEnum()`, `isInner()` and others and
other "properties" of a class.

#### Add transformations

Now it's time to describe how the changed class should look like.

```kotlin
val paramList =
    value.getValueParameters().map { "@kotlinx.serialization.SerialName(\"${it.name?.toSnakeCase()}\") ${it.text}" }
val paramListString = paramList.joinToString(", ", "(", ")")
val newDeclaration = """
                       |$`@annotations` $kind $name$`(typeParameters)`$paramListString {
                       |    $body
                       |}
                   """.trimMargin()
Transform.Companion.replace(classElement, newDeclaration.`class`.syntheticScope)           
```

First, we collect a list of parameters with an added serial name (`val parameterName: String`
-> `@SerialName("parameter_name") val parameterName: String`) and then join them to a single string.

The transformation to snake case is done with an extension function:

```kotlin
fun String.toSnakeCase() = splitToWords().joinToString("_").toLowerCase()
```

Where splitting the camel case is done by a regex:

```kotlin
private fun String.splitToWords() =
    split("(?<=[a-z])(?=[A-Z])|(?<=[A-Z])(?=[A-Z][a-z])|(?<=[0-9])(?=[A-Z][a-z])|(?<=[a-zA-Z])(?=[0-9])".toRegex())
```

Check [this thread](https://stackoverflow.com/questions/7593969/regex-to-split-camelcase-or-titlecase-advanced#comment57838106_7599674)
to get the details.

Second, we redeclare our class and put mapped parameters instead of previous ones.

It's like a puzzle: you take "destructured" blocks of your class and put them back with necessary updates.

To avoid redeclaration of an annotation you may need to add a filter, that checks, if the field is already annotated
with `@SerialName`:

```kotlin
val skip = value.getValueParameters().all { it.isAnnotatedWith(".*SerialName.*".toRegex()) }
```

Now use this condition in your transformation:

```kotlin
Transform.Companion.replace(classElement, if (skip) this else newDeclaration.`class`.syntheticScope)
```

### The function

```kotlin
val Meta.serialNameGenerator: CliPlugin
    get() =
        "Serial Name Plugin" {
            meta(
                classDeclaration(this, { isAnnotatedWith("@Serializable".toRegex()) }) { classElement ->
                    val skip = value.getValueParameters().all { it.isAnnotatedWith(".*SerialName.*".toRegex()) }
                    Transform.replace(classElement, if (skip) this else {
                        val paramList = value.getValueParameters()
                            .map { "@kotlinx.serialization.SerialName(\"${it.name?.toSnakeCase()}\") ${it.text}" }
                        val paramListString = paramList.joinToString(", ", "(", ")")
                        val newDeclaration = """
                            |$`@annotations` $kind $name$`(typeParameters)`$paramListString {
                            |    $body
                            |}
                        """.trimMargin()
                        newDeclaration.`class`.syntheticScope
                    })
                }
            )
        }
```

### Checking the result

Let's check it in a simple code:

```kotlin
fun main() {
    val cat = Cat("Chonk", 3, "Max", "British Shorthair")
    println(Json.encodeToString(cat))
}
```

Output:
`{"name":"Chonk","age":3,"owner_name":"Max","breed":"British Shorthair"}`

Aaaand... it works!

### Summary

This is an example of one of the things that you can achieve with Arrow Meta. As a next step you may want to try to make
the naming strategy plugin more flexible (supporting multiple naming conventions and make it configurable). Be aware
though that Arrow Meta may not be stable enough to use it in production yet.

