---
layout: post
title:  "Context Receivers in Kotlin"
date:   2022-05-18 08:00:00 +0100
categories: Kotlin
author: Fatih Coşkun
---
# Context Receivers In Kotlin

Kotlin [1.6.20](https://kotlinlang.org/docs/whatsnew1620.html) finally introduces 
[context receivers](https://github.com/Kotlin/KEEP/blob/master/proposals/context-receivers.md), a new 
language feature that subsumes several language features that have been hotly discussed on various Kotlin boards for
many years.

In this post we will see what context receivers are, how they compare to other types of receivers in Kotlin, and various
use cases in which they solve problems more concisely than how it was possible before.

> ⚠️  As of today, context receivers are available as a prototype on Kotlin/JVM only. To use them, you will need to
> enable them explicitly via the compiler flag `-Xcontext-receivers`.
>
> Examples shown in this post work with the prototype available in Kotlin 1.6.20, but be aware that future iterations
> might make slight design changes to this feature.

## Other Kinds Of Receivers

Before we will see what context receivers are, we need to understand which kinds of receivers already exist in Kotlin
and how they compare to each other. More specifically, we will see the difference between class members and
extensions.

### Class Members / Dispatch Receivers

Classes in Kotlin can have members, which can either be properties or functions. Class members have the following
characteristics:

* can be called / accessed only by providing a so-called *dispatch receiver*
* the dispatch receiver can be provided explicitly (qualified syntax) or implicitly
* `this` within a member body refers to its dispatch receiver

The following example shows a Person class that has 3 member properties and 1 member functions.
The example also shows how members can be accessed implicitly or explicitly via qualified syntax. 

```kotlin
class Person(val name: String, val birthday: LocalDate) {
  val age get() = birthday.until(LocalDate.now(), ChronoUnit.YEARS)

  fun greet(): String {
    // provides dispatch receiver for "name" property explicitly
    // provides dispatch receiver for "age" property implicitly
    return "Hello, I'm ${this.name}, I'm $age years old"
  }
}
```

Of course members of a class can also be accessed from outside the class itself. And again, the receiver can be provided
either implicitly or explicitly as demonstrated in the following.

```kotlin
fun main() {
  val person = Person("John", LocalDate.of(1980, 1, 1))

  // provides dispatch receiver of greet() function explicitly:
  println(person.greet())

  with(person) {
    // provides dispatch receiver of greet() function implicitly:
    println(greet())
  }
}
```

> ℹ️️ The above example makes use of the `with` [scoping function](https://kotlinlang.org/docs/scope-functions.html), 
> which will make its first argument available as the receiver of its second lambda argument.

### Extensions / Extension Receivers

Kotlin also provides the possibility to declare extension properties or extension functions. They act similar to class
members in some characteristics:

* can be called / accessed only by providing a so-called *extension receiver*
* the extension receiver can be provided explicitly (qualified) or implicitly
* `this` within an extension body refers to its extension receiver

The following example shows a Person class that has 3 extensions. The example shows also how the extensions can be
accessed implicitly or explicitly via qualified syntax. 

```kotlin
class Person(val firstName: String, val lastName: String, val birthday: LocalDate)

val Person.name get() = "$firstName $lastName"
val Person.age get() = birthday.until(LocalDate.now(), ChronoUnit.YEARS)

fun Person.greet(): String {
  // provides extension receiver for "name" extension property explicitly
  // provides extension receiver for "age" extension property implicitly
  return "Hello, I'm ${this.name}, I'm $age years old"
}
```

Again, the extensions of a class can also be accessed from outside the class or its extensions, by either providing the
extension receiver implicitly or explicitly. In fact, the code does not differ from accessing members of a class:

```kotlin
fun main() {
  val person = Person("John", "Peters", LocalDate.of(1980, 1, 1))

  // provides extension receiver of greet() extension function explicitly:
  println(person.greet())

  with(person) {
    // provides extension receiver of greet() extension function implicitly:
    println(greet())
  }
}
```

### Member Extensions

In Kotlin it is possible to declare properties or functions that are members of a class, and at the same time extensions
of another class. They share some characteristics of members and extensions but are also different in some regards:

* member extensions have **both** a dispatch receiver **and** an extension receiver
* the extension receiver can be provided explicitly (qualified syntax) or implicitly
* the dispatch receiver can only be provided implicitly (no qualified syntax)
* `this` within a member extension refers to the extension receiver
* `this@Type` within a member extension refers to the dispatch receiver

The following example demonstrates characteristics of member extensions:

```kotlin
class Club(val name: String, val members: MutableSet<Person> = mutableSetOf())

data class Person(val name: String) {
  fun Club.join() {
    // "this" within a member extension refers to the extension receiver (Club)
    // "this@Person" refers to the dispatch receiver (Person)
    this.members.add(this@Person)
  }

  fun foundNewClub(clubName: String): Club {
    val club = Club(clubName)
    
    // the extension receiver of the "join" member extension is provided explicitly
    // the dispatch receiver of the "join" member extension is provided implicitly
    club.join()
    
    return club
  }
}

fun main() {
  val person = Person("John")
  val club = Club("Kotlin User Group")

  with (person) {
    // the extension receiver of the "join" member extension is provided explicitly
    // the dispatch receiver of the "join" member extension is provided implicitly
    club.join()
  }

  with (person) {
    with (club) {
      // both receivers are provided implicitly
      join()
    }
  }
}
```
We can see in above example that member extensions differ from ordinary members or extensions in one important regard:
Since the syntax does not allow us to provide more than 1 receiver explicitly, we are forced to provide the dispatch
receiver implicitly.

Next, we will see one more example for member extensions that will demonstrate a real use case for them. It is taken from
the Android world.

```kotlin
class MyActivity : Activity() {
  private val Int.dp get() = this * this@MyActivity.resources.displayMetrics.density

  override fun onStart() {
    val bitmap = Bitmap.createBitmap(200.dp, 100.dp)
    // ...
  }
}
```
The example above defines a member extension property `dp` which has 2 receivers. Its extension receiver is an integer,
while its dispatch receiver is an Activity.

Normally, we would like to define this property in some global package as a top level extension property on Int type.
The problem is that the implementation of this property needs both of its receivers, the integer and the
Activity. Prior to Kotlin 1.6.20 there was no way to have top level declarations with more than 1 receiver.  

So now, let's finally see how we can improve things with Kotlin's new context receivers feature.

## Context Receivers

Beginning with Kotlin 1.6.20 we can define contextual functions or properties as well as contextual classes. They have
the following characteristics:

* they can have arbitrary number of *context receivers*
* they can only be called / accessed by providing all of their context receivers
* the context receivers can only be provided implicitly (no qualified syntax)
* `this` within their body never refers to their context receivers
* `this@Type` can be used to refer to their context receivers
 

Of course, one can also define declarations that have a combination of dispatch, extension or contextual receivers.
There are many use cases in which contextual declarations can lead to concise code that eliminates boilerplate. We will
see several of them in the following.

### Use Case: Multiple Receivers

Let's improve our previous example that was taken from the Android world. By using context receivers, we can now finally
define the `Int.dp` extension as a top level declaration.

```kotlin
context(Activity)
val Int.dp: Int get() = this * this@Activity.resources.displayMetrics.density

class MyActivity : Activity() {
  override fun onStart() {
    val bitmap = Bitmap.createBitmap(200.dp, 100.dp)
    //...
  }
}
```

This top level property has both an extension receiver (Int) and a context receiver (Activity). Since the integer is its
extension receiver, we can provide it explicitly, which leads to the convenient access syntax `200.dp`. In addition,
this property has Activity as its context receiver, which means 2 things:
1. within the property's body we can access this context receiver
2. we need to provide this receiver, when calling this property

Since there is no syntax for providing a context receiver explicitly, we need to provide it implicitly. In above
example, it is provided implicitly for us because `onStart` is a member function of an Activity.

> ℹ️️ in above example, we are using `this@Activity.resources` to access the resources property of the context receiver.
> We are doing it here for the sake of understanding what is going on. Once you understand the concept it is perfectly
> fine to access the context receiver's `resources` property implicitly by omitting the explicit `this@Activity`
> reference to the context receiver.

### Use Case: Composable DSLs

In the following example, we are using the `buildJsonObject` function, which is provided by the kotlinx.serialization
library. Its usage is not the most convenient:

```kotlin
val jsonObject = buildJsonObject {
  put("clubName", "Kotlin User Group")
  putJsonArray("members") {
    add(buildJsonObject {
      put("name", "Bob")
      put("age", 12)
    })
    add(buildJsonObject {
      put("name", "Sue")
      put("age", 34)
    })
  }
}
```

We would like to create a DSL that will simplify above example and delegate to the `buildJsonObject` function. Prior
to context receivers this was possible only by using member extensions: create a custom class, that would either
duplicate all members of the `JsonObjectBuilder` class or subclass it (if we are lucky, and it is open). Either way,
lots of code ceremony is involved and the result is still very inflexible.

Thanks to context receivers, this is now very easy to do with top level functions:

```kotlin
fun json(action: JsonObjectBuilder.() -> Unit) = buildJsonObject(action)

context(JsonObjectBuilder)
infix fun String.by(value: String) = put(this, value)

context(JsonObjectBuilder)
infix fun String.by(value: Number) = put(this, value)

context(JsonObjectBuilder)
infix fun String.by(values: List<JsonElement>) = putJsonArray(this) {
  for (value in values) add(value)
}


val json2 = json {
  "clubName" by "Kotlin User Group"
  "members" by listOf(
    json {
      "name" by "Bob"
      "age" by 12
    },
    json {
      "name" by "Sue"
      "age" by 34
    }
  )
}
```

### Use Case: Dependency Injection

Consider the following simple `SearchController`. It is supposed to be managed by a dependency injection framework,
that will inject 2 dependencies for us, a `Urls` object and an `I18n` object.

This SearchController is rendering a very simple HTML page, and for doing so, it is delegating parts of its
implementation to a global top-level `homebreadcrumb` function.

```kotlin
interface Urls {
  val homeUrl: String
  val searchUrl: String
}

interface I18n {
  val homeTitle: String
  val searchTitle: String
}

class SearchController(
  private val urls: Urls,
  private val i18n: I18n,
) {

  fun render(): String = """
    <html>
      <head>
        <title>${i18n.searchTitle}</title>
      </head>
      <body>
        ${homeBreadcrumb(urls, i18n)}
        <h1>${i18n.searchTitle}</h1>
      </body>
    </html>
  """.trimIndent()
}

fun homeBreadcrumb(urls: Urls, i18n: I18n): String = """<a href="${urls.homeUrl}">${i18n.homeTitle}</a>"""
```

The problem in above example is that dependency injection does not play well with top level functions. The `urls` and
`i18n` dependencies cannot be injected automatically into the `homebreadcrumb` function for us. Instead, we need to 
provide them explicitly as arguments. This can quickly get out of hands, if the number of dependencies is large.

Prior to Kotlin 1.6.20 there was a way to solve this with a pattern that we call the *contextual interface pattern*.
Lets see how it looks like:

```kotlin
interface UrlContext {
  val urls: Urls
}

interface I18nContext {
  val i18n: I18n
}

class SearchController(
  override val urls: Urls,
  override val i18n: I18n,
) : UrlContext, I18nContext {

  fun render(): String = """
    <html>
      <head>
        <title>${i18n.searchTitle}</title>
      </head>
      <body>
        ${homeBreadcrumb()}
        <h1>${i18n.searchTitle}</h1>
      </body>
    </html>
  """.trimIndent()
}

fun <T> T.homeBreadcrumb(): String where T : UrlContext, T : I18nContext {
  return """<a href="${urls.homeUrl}">${i18n.homeTitle}</a>"""
}
```
The main difference to the previous example is that our new `SearchController` now implements 2 context interfaces, the
`UrlContext` and the `I18nContext`. Furthermore, the new `homeBreadcrumb` function is now defined as an extension on a
generic type `T` which needs to be a subtype of both of those context interfaces.

Because our new controller class is implementing both of those context interfaces, it can serve as the extension 
receiver of our new `homebreadcrumb` function. The dependencies do not anymore exist as arguments, and the extension
receiver is conveniently provided implicitly.

Besides the slightly more complex signature for our top level function, the main disadvantage of this pattern is that
we now need to make both of the controller's dependencies *public* properties of the class.

We can solve both of these disadvantages by making both the `SearchController` class and `homebreadcrumb` function have
context receivers:

```kotlin
context(UrlContext, I18nContext)
class SearchController {
  fun render(): String = """
    <html>
      <head>
        <title>${i18n.searchTitle}</title>
      </head>
      <body>
        ${homeBreadcrumb()}
        <h1>${i18n.searchTitle}</h1>
      </body>
    </html>
  """.trimIndent()
}

context(UrlContext, I18nContext)
fun homeBreadcrumb(): String = """<a href="${urls.homeUrl}">${i18n.homeTitle}</a>"""
```


### Use Case: "Type Classes"

Consider the following 3 functions, which have a very similar implementation. In fact there is some code duplication
going on in them.

```kotlin
fun List<Int>.sum(): Int = fold(0) { acc, e -> acc + e }

fun List<String>.sum(): String = fold("") { acc, e -> acc + e }

fun <T> List<List<T>>.sum(): List<T> = fold(emptyList()) { acc, e -> acc + e }
```

Interestingly, the `acc + e` expression in each of them is not part of the duplication. In each of those functions
the plus operator resolves to a different standard library function (integer addition, string concatenation and list
concatenation).

So what exactly is duplicated? We will answer this question by reverting the question: the only thing
that is *different* in each of those functions is the initial *neutral* value and the operator to use to combine the 
`acc` and `e` arguments.

Can we create an abstraction for this, such that we can use the algorithm in more use cases? In Kotlin 1.6.20, one very
convenient way to do this is using context receivers in a way that resembles what's called *Type Classes* in other
languages.

```kotlin
interface Monoid<T> {
  val unit: T
  operator fun T.plus(other: T): T
}

context(Monoid<T>)
fun <T> List<T>.sum(): T = fold(unit) { acc, e -> acc + e }
```
Instead of implementing the algorithm 3 times, we now implement it only once on a list of arbitrary generic type `T`.
In addition, we declare that the function has a context receiver of type `Monoid<T>` (aka the type class). The monoid 
interface provides us the parts that need to be abstracted: the initial neutral value for our folding algorithm is 
encoded in the monoid's `unit` property, and the combination operation is represented by the monoid's `plus` function.

Now we can define concrete implementations of our monoid interface for our 3 use cases and use the sum function like
this:

```kotlin
object IntMonoid : Monoid<Int> {
  override val unit = 0
  override operator fun Int.plus(other: Int) = this + other
}

object StringMonoid : Monoid<String> {
  override val unit = ""
  override operator fun String.plus(other: String) = this + other
}

class ListMonoid<T> : Monoid<List<T>> {
  override val unit = emptyList<T>()
  override operator fun List<T>.plus(other: List<T>) = (this as Collection<T>) + other
}

fun List<Int>.sum() = with(IntMonoid) { sum() }
fun List<String>.sum() = with(StringMonoid) { sum() }
fun <T> List<List<T>>.sum() = with(ListMonoid<T>()) { sum() }
```

Now our API can not only be used for these 3 use cases, but we enabled the users of our API to provide their own
implementations of the monoid interface, making it applicable for arbitrary use cases.

> ℹ️️ To be precise, there is a subtle difference between Kotlin's Context Receivers and Type Classes in other languages.
> As we can see in above example, at some point we are forced to provide a concrete Monoid instance 
> `with(IntMonoid) { sum() }`. Other languages that support type classes usually resolve the type class instance
> automatically.

### Use Case: Context Oriented Programming

In a broad sense all the previous use cases have been context oriented programming. In a more narrow sense this is a
rather new 
[programming paradigm](https://proandroiddev.com/an-introduction-context-oriented-programming-in-kotlin-2e79d316b0a2), 
which will probably lead to even more use cases.

One such use case can already be seen when it comes to library development. Whenever a library requires its users to
chain an argument object through a call chain, without requiring the user to take actions on the object themselves, the
object could now become a context receiver.

Examples for such objects are the CoroutineScope or database transactions.

## Outlook
The context receivers feature is currently heavily [discussed](https://github.com/Kotlin/KEEP/issues/259), which might
lead to adjustments to its current [design](https://github.com/Kotlin/KEEP/blob/master/proposals/context-receivers.md).
Furthermore, there are a couple of very exciting followup features that play very well with
context receivers like
[scope properties](https://github.com/Kotlin/KEEP/blob/master/proposals/context-receivers.md#scope-properties) or
[decorators](https://github.com/Kotlin/KEEP/blob/master/proposals/context-receivers.md#future-decorators).

The Kotlin developers are inviting the community to play with the prototype implementation and provide feedback on the
official [KEEP discussion thread](https://github.com/Kotlin/KEEP/issues/259).
