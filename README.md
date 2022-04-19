# Note

The following notes were compiled from the courses available in udemy:

[Part1](https://www.udemy.com/course/scala-advanced-part-1-the-scala-type-system)
[Part2](https://www.udemy.com/course/scala-advanced-part-2)
[Part3](https://www.udemy.com/course/scala-advanced-part-3-functional-programming-performance)

# Scala Type System - Part 1

## Properties and State

Principle of Uniform Access: All services offered by a module should be available through a uniform notation, which does not betray whether they are
implemented through storage or through computation.

Scala provides a way to create a method when we want to create a method to set a property like this `person.name = "Jorge"`, for this we need to
define the method like `def name_=(name:String):Unit`.

Scala uses re-writing rules for mutable fields and modifiers:

    * Constructor parameters without val or vars are private to the instance of the class instantiated.
    * A field or parametric field like val foo: Int = 0 gets re-written to:
        private[this] val _foo: Int = 0  // must use different name
        def foo: Int = _foo              // So it creates a getter for the name with the same property name
        def foo_=(d: Int): Unit = { _foo = d }
    * A modifier of the form `baz.bar = 1.0` is re-written by the compiler to `baz.bar_=(1.0)`
    * Abstract or traits classes would get the getters and setters like before, but without implementation

The principle of referential transparency in functional programming states that if you call the same method with the same arguments you should always
get the same results. This is no longer the case with mutability, so extra care needs to be taken. A common scenario for mutable state in scala is for
caching, try to use google guava with futures:

```scala
import com.google.common.cache.{CacheLoader, CacheBuilder}
import scala.concurrent._
import duration._
import ExecutionContext.Implicits.global

object FakeWeatherLookup {
  private val cache = CacheBuilder.newBuilder().
    build {
      new CacheLoader[String, Future[Double]] {
        def load(key: String) = Future(fakeWeatherLookup(key)) // This avoids key lock
      }
    }

  def apply(wxCode: String) = cache.get(wxCode)
}
```

## Type System part 1

A parameterized type let's say `B[T]` is a type of its own, different that let's say `B[Z]`. A generic type can be upper bounded, like for example:
`case class Bowl[F <: D]` which is read like F as long as F is a subtype of D, F must be lower in the inheritance hierarchy than D, and we can treat F
like a D in this particular case. Because `B[T]` is different than `B[Z]`, a method that accepts `B[T]` can't receive any other parametric type unless
the B class is defined as `B[+T]` which is known as covariance (this can also be upper bounded like `B[+T <: D]`).

Contravariance happens when given `B` subtype of `C`, `T[C]` is a subtype of `T[B]`. This comes handy when you have a situation when you require some
generic to be less specific on the type parameter it accepts, i.e:

```scala
trait Sink[-T] {
  def send(item: T): String
}

object AppleSink extends Sink[Apple] {
  def send(item: Apple) = s"Coring and eating ${item.name}"
}

object FruitSink extends Sink[Fruit] {
  def send(item: Fruit) = s"Eating a healthy ${item.name}"
}

def sinkAnApple(sink: Sink[Apple]): String = {
  sink.send(Apple("Fuji"))
}
sinkAnApple(AppleSink)
sinkAnApple(FruitSink) // This is allowed because Sink is contravariant on T
```

Important notes:

    * Any supplied type T is invariant, covariant and contravariant to itself!
    * Only type T is all three of these things to type T
    * Any type T also satisfies lower and upper bounds to itself
    * Class/Trait type parameters are the only place you can use variance modifiers:
          1 - not in method type parameters
          2 - not in any usage of a type parameter
    * We often say a class or trait is co- or contra-variant, but in fact type parameters, not the containing types have variance.
    * And a class or trait can have both co- and contra-variant type parameters in its definition

As a general rule of thumb, a class or method that accepts a type parameter can accept a more general type than it defines (Contravariance, and return
a more specific type than it defines (covariant).

### Variance Problems

Imagine the trait:

```scala
trait CombineWith[+T] {
  val item: T

  def combineWith(another: T): T
}
```

The problem with the above is that it allows us to do something like:

```scala
val cwo: CombineWith[Any] = CombineWithInt(10)
cwo.combineWith("ten")
```

Which returns the following cryptic compilation error _covariant type T occurs in contravariant position in type T of value another ..._. So how do we
solve this problem? By introducing Bounds to the type:

```scala
case class FoodBowl[+F <: Food](food: F) {
  override def toString: String = s"A bowl of ${food.name}"

  def mix[M >: F <: Food](other: M) = MixedFoodBowl[M](food, other)
}
```

In the above example, the type M is the most specific supertype for all items in the new List that is also a subtype of Food. This is called the Least
Upper Bounds (or sometimes LUB).

## Type System part 2

A type parameter is part of the overall type. But the type parameter can be used from outside the generic container:

```scala
class Foo[F](i: F) {
  val theX = x
}

val a: Foo.F // we can't do this
```

We can have types as member of a class to solve the above problem:

```scala
abstract class Foo2 {
  type F <: Foo
} // F is a generic field now, not a generic parameter and we can use method of the class F

class Foo3 extends Foo2 {
  type F = Int
}

val theClass = new Foo3()
val a: theClass.F = 10 // now we can use the type
val b: Foo3#F = 20 // equivalent to the above
```

Every Scala instance has a unique type member associated with it --> `.type` which is a Singleton type.

```scala
val h = "hello"
def say(greet: h.type) = print(h)
say("b") // this won't compile, we can only pass say(h), this is helpful to define DSLs
```

A Technique to avoid to define type parameters when creating an instance is the following:

```scala
object Foo {
  def apply[T <: FOO](f: T) = new Foo() {
    type ALIAS = T
    override val innerAttr: T = f
  }
}

val fooInstance = Foo(new OtherClass()) // The type is Foo{Type ALIAS = OtherClass}, no need to define the type like val fooInstance: Foo[OtherClass]
```

The above technique is known as path dependent type because the way you access the type defines it. The inner type in two different instances of the
same class is different, the type contains the instance definition of it.

    * Type members available from inside and outside of the class or trait
    * Type parameters only available from inside of the class
    * Type parameters become part of the type and must be specified
    * Type members are enclosed fully in a new type
    * Type parameters can have co- and contra-variance: `trait FruitBowl[+F <: Food]`
    * Type members cannot have variance, and use bounds instead: `trait FruitBowl2 { type FOOD <: Food }`

Recursive types are types that refers to themselves, like `class B{ def compareTo(b:B)} // We can use B eventhough it hasn't be fully defined`. But
what happens when we introduce generics? Then we get into recursive types, for example:

```scala
trait CompareT[T] {
  def <(other: T): Boolean
}

// Here the T must implements (is upper bounded) the trait CompareT, any T that implements it, can be passed
def genInsert[T <: CompareT[T]](item: T, rest: List[T]): List[T] = rest match { // [T <: CompareT[T]] becomes part of the overall type
  case Nil => List(item)
  case head :: _ if item < head => item :: rest // this is possible because T implements CompareT
  case head :: tail => head :: genInsert(item, tail)
}

def genSort[T <: CompareT[T]](xs: List[T]): List[T] = xs match {
  case Nil => Nil
  case head :: tail => genInsert(head, genSort(tail))
}
```

One of the issues with the above is that you need to pass the type definition `[T <: CompareT[T]]` to all methods that uses that type. So a class
definition that satisfies the above needs to be defined like `class Foo(x:Int) extends CompareT[Foo]{...}`, which is a common pattern in Scala know as
_F-Bounded Polymorphism_ (Since the type T refers to itself in the definition, it is also called a Recursive Type).

## Type System part 3

### Existential types

In scala, it is mandatory to define the type of the arguments, this will not compile `def someMethod(xs:list) = {...}`, instead we need to provide
information about its type, even if it is just a generic `def someMethod[T](xs:list[T]) = {...}`. However, there is a way to circumvent this by using
an _existential type_ which allows to skip the definition of a type parameter: `def someMethod(xs:list[T forSome {Type T}]) = {...}` (to use this
feature you might need to import `language.existentials`), or use the much concise form `def someMethod(xs:list[_]) = {...}`. Existential types can
also be bounded like in `def someMethod(xs:list[T forSome{Type T <: B}]) = {...}` or `def someMethod(xs:list[_ <: B]) = {...}`.

### Structural types

You can also use something called structural type, which is a way to define an ad hoc type, for example:
`def someMethod(xs:list[_ <: {def size():Int}]) = {...}`, which is a way to say, the type hold by this collection needs to implement the method size.
This is also known as static duck typing, and uses reflection, so be careful when using it. This is useful for shortcut actual reflection, for example
if we know an instance of _AnyRef_ is of type String, we can call the length method of it like `obj.asInstanceOf[{def length:Int}].length`.

### Refinement types

Imagine that we want to create a method that accepts a parameterized type, and we want to restrict the method to use only a specific parametric type,
can we create something similar for classes that has type members instead of parameterized types? We can with the so called refinement types. For
example:

```
abstract class Parent {
  type NAME
}

class Son extends Parent {
  type NAME: String
}

class Daughter extends Parent {
  type NAME: Int
}

// We can use a variation of a structural type to restrict the type member type.
def printTheName(name: Parent {type NAME <: String})
```

This enforces extra rules at compile time, no reflection is invoked.

### Self types

Imagine we have an embedded class like the following:

```scala
class A(name: String) {
  class B(name: String) {
    override def toString(): String = s"$name vs $name"
  }
}
```

The above method will print B name twice, we can add an alias on A like `class A{outer => ...` so we can refer to A's name like `outer.name`, outer is
an alias to A (the default alias is `this`). We can also narrow the type down of the alias, for example we can define a trait like:
`trait B{self:Ordered =>...}`, whomever uses this trait must provide Ordered as well, this is different than extending the class Ordered, it simply
specifies dependency not hierarchy.

### Infix type notation

In scala, you can call a method like `a.+(b)` or using the infix notation you can call it also like `a + b`. Something like this can be used with
parameterized types with two type parameters, for example `Or[A,B]` can be substituted like `val a: A Or B`. One thing to note is that if you use this
in a class definition, you need to enclose it in parentheses: `class C extends (A Or B){...}`.

### Scala Enumerations

Enumerations in scala are defined using path dependent types, you need to define an object that extends `Enumeration`, define the values inside the
object and assign them to `Value`, which internally increments a counter by 1 everytime the function is called:

```scala
object Colors extends Enumeration {
  val Red, Green = Value
}
```

So what happens here is that although Value is a type Integer, because is defined as a path dependant type inside Color, any other Enumeration which
has a type member with an integer value of 1 won't be the same as the value 1 inside color because they are different types. Enumerations are limited
in scala, so usually sealed traits are used instead.

## Implicits part 1

Scala is a statically typed language, therefore adding classes specific for certain purposes (like counting the number of retries of an operation)
reduces ambiguity and it is a desirable thing to have. Rules that governs implicits:

    * Implicits are looked up by the Scala compiler when required
    * If code will compile without an implicit, Scala will not look one up or apply it
    * Implicit parameter lists can have more than one parameter
    * Only the final parameter list can be implicit
    * If any implicits are explicitly provided, all of them must be provided
    * implicit is only placed at the head of the parameter list, not on each parameter

The scala compiler cares about the type of the implicit to fill it in, not about the name.

### Type Classes

A type-class is another name for a trait or abstract class with a single generic type parameter and abstract method(s) defined for that type. Given a
typle class `TheClass[T]`, because we know nothing about T, we can't call methods on it, but we can call methods on the `TheClass` class.

### Providing implicit type classes

If we pass an implicit parameter to a method, we make it available for other methor that might require that implicit parameter within the scope of the
method. Implicits provides a secondary type system in Scala, orthogonal to the primary one. By leveraging implicits, I can provide functionality
without changing the inheritance (the implicit type might even be in a provided library). If you want to define an implicit implementation for a type
on the spot, you can do the following:

```scala
implicit val theImplicit = new Comparable[Int] {
  ...
}

// Replace the above by the more idiomatic form below:
implicit object TheImplicit extends Comparable[Int] {
  ...
}
```

Vals, objects, classes and def can be marked as implicit.

### Ad-hoc Polymorphism

We can effectively add behavior to types without needing to extend other types even for types we don't own. This capability is known as ad-hoc
polymorphism. Scala is even more powerful in that you can choose which implicits will be imported/used.

### Context Bounds

If an implicit parameter is required but never explicitly used, like in:

```scala
def genSort[T](xs: List[T])(implicit cmp: CompareT[T]): List[T] = xs match {
  case _ => genInsert(head, genSort(tail))
}
```

You can replace this and use _context bounds_, the declaration of the method would then be `def genSort[T: CompareT[T])(xs: List[T]): List[T]{...}`,
which can be read by type `T` where `Compare[T]` is available. If for whatever reason you need to make use of the implicit parameter, you can do
something like this in the body of the method that uses the parameter: `val cmp = implicitly[TheClass[T]]`, and cmp would be assigned the implicit
used.

We said before that def can be implicit too, and defs can take parameters, which can't be filled when declaring the implicits, but if the implicit def
has implicit parameters, the scala compiler will look for those parameters and fill them as well, which is a very powerful feature:

```scala
// We define the contract a class should implement
trait JSONWrite[T] {
  def toJsonString(item: T): String
}

// This method requires a JSONWrite to be available so we can jsonify it
def jsonify[T: JSONWrite](item: T): String = implicitly[JSONWrite[T]].toJsonString(item)

// We make available a JSONWrite for string
implicit object StringJSONWrite extends JSONWrite[String] {
  def toJsonString(item: String) = s""""$item""""
}
// So we can call the method for the type String, or any other implementation for the type passed
jsonify("The method works")
```

We can push the above a but further and accept also type classes, for example `List[T]`. The problem is that we don't want to define an implicit for
the class `List[String]` another for `List[Int]` and so on. We can define an implicit for `List[T]` and work with any list, as long as there is an
implicit defined for that generic type `T`:

```scala
implicit def listJSONWrite[T: JSONWrite] = new JSONWrite[List[T]] {
  def toJsonString(xs: List[T]): String = {
    xs.map(x => jsonify(x)).mkString("[", " ,", "]")
  }
}
```

### Implicit rules and restrictions

    * Only one parameter list can be implicit, and it must be the last parameter list
    * The implicit parameter list starts with the implicit keyword
    * All parameters in the list are then implicit
    * Parameters may be supplied explicitly (but if so, all of them must be supplied)
    * Implicit values, objects or defs must be imported directly into scope by name (not accessible through dots)
    * Scala will not apply an implicit if the code will compile without doing so

### ClassTags and TypeTags

#### ClassTag

Inserts a `classOf` for a generic type when used as an implicit bound: `def someFunction[T: ClassTag](x: T)` which gets us a Class type representing
the type T. We can get the ClassTag by calling the `classTag[T]` method within the scope of `someFunction`. A ClassTag can be used on pattern match:

```scala
import scala.reflect._

def isA[T: ClassTag](x: Any): Boolean = x match {
  case _: T => true
  case _ => false
}
```

#### TypeTag

ClassTag only provides class tagging (and pattern matching) for the top type, but TypeTag provides almost everything the Compiler knows about the
type. So the above method will fail if x is a `Map[String, Int]` because it will have information about the Map type, but not the String or Int
parameter types.

```scala
import reflect.runtime.universe._

val tt = typeTag[Map[String, Int]]
val theType = tt.tpe // With the above we can call the following:
theType.baseClasses // returns all the base classes for the Map in this case
theType.typeArgs // returns List(String, Int)
```

With this `typeTag` we can do pattern match using guards and the `=:=` method of the TypeTag class:

```scala
case class Tagged[A](value: A)(implicit val tag: TypeTag[A]) // we drag the TypeTag along with the case class

def taggedIsA[A, B](x: Tagged[Map[A, B]]): Boolean = x.tag.tpe match {
  case t if t =:= typeOf[Map[Int, String]] => true
  case _ => false
}
```

## Implicits part 2

Type parameters become a part of the overall type and implicit parameters matches and fills in a unique type. Therefore, availability of an implicit
for multiple types can enforce rules. Imagine we want to enforce the existence of a _FOOD_ which corresponds to an _EATER_:

```scala
implicit object VeganEatVegetables extends (Vegan Eat Vegetables) // this makes the below compile

def feedTo[FOOD <: Food, EATER <: Eater](food: FOOD, eater: EATER)(implicit ev: Eats[EATER, FOOD]) = {
  ev.feed(food, eater)
}
```

The existence of the implicit evidence guarantees that we can only use this method if there is an implicit Eats available for the food and eater type.
That's what it is known as compile-time enforced behavior.

### Method Imposed Type Constraints

Imagine we want to constraint the use of certain methods (for example '*' on a cell in a spreadsheet only for numeric values) to certain types. We can
create a context bound by the type we want to enforce and ask Scala to prove that this new type for which we want to verify the operation can be
applied is the same as T:

```scala
case class Cell[T](item: T) { // We define the cell for the type T
  def *[U: Numeric](other: Cell[U])(implicit ev: T =:= U): Cell[U] = { // Then we require that there is implicit evidence that given another cell
    val numClass = implicitly[Numeric[U]] // of type U (let's say a Double), there is implicit evidence that there is
    Cell(numClass.times(this.item, other.item)) // numeric for our type T.
  }
}
```

### Safe Casting

Imagine that we want to flatten a `Future[Future[T]]` for any given type T, we can do something like:
`def flatten[F](implicit ev: T =:= Future[F]): Future[F] = FlattenFuture(this.asInstanceOf[Future[Future[F]])`, we can do this safe casting because we
have proved that T is a `Future[F]` with the `=:=`. This requires the type parameter to be provided in use:
`val flattened: Future[String] = futureFutureString.await[String]` (not that we need to parameterize the await method). This is because the type
parameter will be inferred as Nothing before the implicit evidence is found, and hence the implicit evidence will not be found but using `<:<` fixes
this problem:
`def flatten[F](implicit ev: T <:< Future[F]): Future[F] = FlattenFuture(this.asInstanceOf[Future[Future[F])`
Variance in `<:<` allows the implicit to be found, then Scala infers the more appropriate type from there.

### Implicit Conversions

When Scala has a type problem between two types, it looks for an implicit conversions to/from either type that will solve it. You can theoretically
convert any type to any other type implicitly, but as a rule of thumb, you should own at least one of such types. Implicit classes make this easy (and
require no language import).

```scala
implicit class TimesInt(i: Int) {
  def times(fn: => Unit): Unit = {
    varx = 0
    while (x < i) {
      x += 1
      fn
    }
  }
}
```

Effectively, Scala generates the implicit method for you automatically `implicit def TimesInt(i: Int): TimesInt = new TimesInt(i)`. This feature
requirea a wrapping at runtime into the target class. Implicit value classes avoid this overhead:

```scala
implicit class TimesInt(val i: Int) extends AnyVal {
  /*Code here*/}
```

Must have just one public parametric field and no other state, and extend AnyVal. Methods are now generated as static methods, no runtime wrapping
required.
Implicits should be placed in:

    - The class you need it
    - A utility object you can import from
    - A package object
    - The class companion object (always searched last but imported and can't be un-invited, so be careful if this is not the default behavior)

Another set of best practices for implicits:

    - Never define implicits between two classes you don't own at least one of
    - Only use companion object implicits when it is always desired/safe
    - Don't over-use implicits, use them where they make sense
    - Also don't avoid implicits, they are the first, safest tool for extending the Scala type system
    - Smart use of implicits can often negate the need for macros (particularly type-classes)
    - Know and apply the rules of implicits

# Scala Type System - Part 2

## Modularization and dependency injection

### Static cling

It is a java term coined to describe inflexible resource dependencies caused by static methods. In scala this happens for example when properties are
hardcoded in objects and used later to create resources like a DB connection and so on.

    * Objects are initialized when they are first referenced
    * Unlike classes, there are no constructor parameters
    * Singletons are the enemy of flexible configuration

We can prevent this problem simply by putting those hardcoded variables as constructor arguments to classes instead of objects. The problem is that
this can get pretty large the chain of parameters to set in case we start to compose classes and services.

### Runtime Dependency Injection

Maybe we can use dependency injection with annotations, but the problem of these frameworks (like guice) in scala is:

    * It's run time and cannot be verified at compile time
    * The syntax is kind of awkward
    * We can do better in Scala without needing any libraries

### Cake

Cake pattern is called like that because it composes an application by adding several layers on the application by using self traits and abstract
members.

```scala
abstract class DBAccess {
  def database: Database // abstract definition

  def findAgeOfPerson(name: String): Int =
    withTransaction(database.currentSession) { tx =>
      tx.find(name = "Fred").select("age")
    }
}

trait MySQLDB {
  this: DBAccess => // self type
  val database = MySQLDatabase // provide concrete database
}

val access = new DBAccess with MySQLDB // Inject
```

### Parfait

Start with pure abstract traits and various concrete implementations, and then you group the configurations together and that's what you pass down to
classes to be instantiated:

```scala
trait DBConfig {
  val db: Database
}

trait WSConfig {
  def ws: WeatherService
}

trait MySQLDB extends DBConfig {
  lazy val db = MySQLDatabase // provide concrete database
}

trait WeatherUnder extends WSConfig {
  def ws: WeatherService = new WeatherUndergroundRest
}

class DBAccess(implicit config: DBConfig) { // Only DBConfig related values are passed. Note the implicit!
  def database: Database = config.db

  def findZip(city: String): Int = withTransaction(database.currentSession) { tx =>
    tx.find(name = city).select("zip")
  }
}

class WeatherAccess(implicit config: WSConfig) {
  def weatherService: WeatherService = config.ws

  def findTempForZip(zip: String): Float = ???
}

class DoItAll(implicit dbWsConfig: DBConfig with WSConfig) {
  val dbAccessor = new DBAccess // gets config implicitly

}

implicit object StandardConfig extends MySQLDB with WeatherUnder // We wire all together, by providing the implementation of the implicits
```

The above can start to get complicated when you chain several of those configs together, so there is an alternative which is to group those
configurations:

```scala
trait AllConfig extends DBConfig with WSConfig

class DoItAll(implicit allConfig: AllConfig) {
  // ...
}

implicit object StandardConfig extends AllConfig with MySQLDB with WeatherUnder // We need to provide the specifics for the traits
```

All of this is compile time checked, which is also easier to test because you can change the specific of the traits just when using inside the test.

## Best Practices and Idioms

    * Recursive functions and pure functions are faster than `for..yield` loops because these are expanded to `flatMap` and `map` functions
    * If you mix collections and options in a for expression, call .toSeq on Options to get a collection as a return so you mix collections only

### For expressions

Here is an example of a for expression:

```scala
val forLineLengths =
  for {
    file <- filesHere
    if file.getName.endsWith(".sc")
    line <- fileLines(file)
    trimmed = line.trim
    if trimmed.matches(".*for.*")
  } yield trimmed.length
```

The four Gs of for expressions:

    * Generator: file <- filesHere, denoted by the <-, all generators in the same for block must be of the same type (e.g. a collection, or a Try)
    * Guard: if file.getName.endsWith(".sc"), guards short-circuit the for expression, causing filtering behavior
    * inline assiGnment: trimmed = line.trim, a simple val expression that may be used in both the remainder of the for block, and the yield block
    * Give (or the yield), after the yield keyword comes the payoff of the for block setup

For expressions can also be used for asynchronous programming, like in:

```scala
val f1 = Future(1.0)
val f2 = Future(2.0)

for {
  v1 <- f1 // We need to de-reference the future values
  v2 <- f2
} yield v1 + v2
```

### Untying Unit

Any expression resulting in Unit must have a side effect to do anything useful, a parameterless function with side effects should always be called
using parentheses.

### Try and Either and Or

Either differs with Try in that it is more specific on the types, you need to specify both types on the declaration, while Try is a Throwable or
another type. A for projection on an Either type would do it on the Right projection of it.

An `org.scalastatic.Or` is mapped on the left projection, uses infix notation like in `Int Or NumericException`. The left projection is created with
the `Good(SomeType)` and the right one is created with `Bad(SomeType)`. The `Or` differs in the Either, in that it accumulates the results and the
failures (an applicative). Check [this page](http://www.scalactic.org/user_guide/OrAndEvery) for usage.

### Some useful tips

    * Case classes offer much for free, and are often preferable to just using tuples (has field names as well as types)
    * Replace nested ifs by match/case with guards, you can even combine it with case classes to extract specific values
    * If you are receiving a function in a parameter list, consider currying it at the end as it help with the syntax
    * Favor implicit classes over implicit conversions when possible, or use AnyVal classes if you can
    * By-name functions can be accidentally called if you forget to put the symbol `=>` like `doSth(fn: Unit)` instead of `doSth(fn: => Unit)` or
      even replace it with the explicit `doSth(fn: () => Unit)`

## Patterns in Scala

### Algebraic Data Types (ADTs)

It is an abstract supertype with a finite number of define subtypes, which in scala is implemented by sealed traits. This is particularly useful when
we want to have a tight control in pattern matching for example. ADTs can be recursive, like the LinkedList, where the class is defined in terms of
itself.

### Loan Pattern

Scala solution to resource management, like in:

```scala
def withFileLines[A](file: File)(fn: Seq[String] => A): A = // Note the curly parameter to have a nicer syntax
  try {
    fn(Source.fromFile(file).getLines().toList)
  } finally source.close()
```

### Trait Pattern

This pattern is nicely done in scala by mixin traits, like in `class Myclass extends LazyLogging`, but we can go further with this example as it is
selfless (the trait is fully contained with no dependencies or undefined behavior), so we can do something like this instead:

```scala
object LazyLogging extends LazyLogging // a companion object with the implementation (no other dependencies on scope)

import LazyLogging._ // We import the behaviour, no need to extend it on my class to get the functionality, no need to mixin
```

### Stacking traits

Imagine you want to modify the behavior of an underlying functionality of a trait, without needing to implement all of the methods. For this you use
the `abstract override def...` declaration to override the abstract method you want to enhance and make a call to `super.yourMethod`. Such calls are
illegal for normal classes, because they will certainly fail at run time. For a trait it will work so long as the trait is mixed in after another
trait or class that gives a concrete definition to the method.

```scala
trait Logger {
  def message(msg: String): String

  def info(msg: String): Unit = println("INFO: " + message(msg))
}

trait StandardLogger extends Logger {
  override def message(msg: String): String = msg
}

trait DateLogger extends Logger {
  abstract override def message(msg: String): String = s"${LocalDateTime.now()}: ${super.message(msg)}" // Note the call to super
}

class Demo3 extends StandardLogger with DateLogger {
  def testLogging(): Unit = info("Thisisaninfo")
}
```

### Poor Man's interface injection

The ability to compose pure abstract traits with classes that satisfies the class signature at runtime:

```scala
class Door {
  def close(): Unit = println("SLAM!")
}

def closeAll(items: Seq[Closeable]): Unit = for (item <- items) yield Try(item.close())
// note that Door is not Closeable, but I can do this:
val door1 = new Door with Closeable
```

### Streamlined Reflection

Use scala structural types to replace convoluted reflection code:

```scala
"Some String".getClass.getMethod("charAt", classOf[Int]).invoke("Some String", new Integer(1)).asInstanceOf[Char]
// The scala way:
"Some String".asInstanceOf[ {def charAt(i: Int): Char}].charAt(1)
```

### Type Classes

One of the most common patterns. Provides "ad-hoc" polymorphism, or a way of providing behavior for a class without needing to affect the inheritance
hierarchy of the class. The implicit keyword can go in front of objects, defs, classes, vals... which can be used with the type classes to create new
functionality without having to affect the hierarchy of your classes:

```scala
implicit def listJsonWriter[T: JSONWrite]: JSONWrite[List[T]] = // Context bound, we guarantee that there is an implicit JSONWrite[T]
{ (xs: List[T]) => xs.map(implicitly[JSONWrite[T]].toJsonString).mkString("[", ",", "]") }
```

### Auto Case Classes

What if we want to add functionality like jsonify a case class? We can do the following, create an abstract class like this:

```scala
abstract class CaseClassAbstractJsonWriter[T: TypeTag] extends JSONWrite[T] {

  val tt = typeTag[T] // This is to make it performant, as it is only invoked once, and not per class
  // This abstract class is going to be mixed in the case class companion object, so by adding this implicit, the compiler will look in the
  // companion object to resolve the implicit type, so the implementation of the writer can be found on the case class itself implicitly
  implicit val writer: JSONWrite[T] = this
}
```

Then we need to make use of some reflection capabilities in scala, we need to introduce a case class writer for every arity of case class that we are
going to support (usually up to 22):

```scala
abstract class CaseClassJsonWriter2[A: JSONWrite, B: JSONWrite, T: TypeTag] extends CaseClassAbstractJsonWriter[T] {

  def unapply(x: T): Option[(A, B)] // we know case classes come with an unapply method to extract the components is made of

  private val aJson = implicitly[JSONWrite[A]]
  private val bJson = implicitly[JSONWrite[B]]


  private val name = this.getClass.getSimpleName.filterNot(_ == '$') // this is going to be mixin in the companion object, which has a $ at the end

  // The copy method of a case class can be invoked with one to all parameters, so we grab the name of the parameters to infer the property names
  // of the case class
  private val fieldNames = tt.tpe.member(TermName("copy")).info.paramLists.flatMap(_.map(_.name.toString))

  // we get the values of the class by doing unapply and then we 'jsonify' the values
  private def fields(x: T): List[String] = unapply(x) match {
    case Some((a, b)) =>
      List(
        aJson.toJsonString(a),
        bJson.toJsonString(b)
      )
    case None => throw new IllegalStateException("Cannot serialize")
  }

  // We join the field names and its values already 'jsonified' and separate the values by comma
  override def toJsonString(item: T): String = {
    val fieldPairs = fieldNames.zip(fields(item))

    val fieldStrings = for ((name, value) <- fieldPairs) yield {
      s""""$name": $value"""
    }

    val allFields = fieldStrings.mkString(", ")
    // Finally the name of the class in conjunction with its fields and values
    s""""{
       |  "$name": {$allFields}
       |}""".stripMargin
  }
}
```

### DSLs and fluent APIs

Scala is so flexible that allows you to do stuff like this, which allows us to write very readable DSL's (scala match actually uses a similar
approach):

```scala
val or: String = "or"
val to: String = "to"

object To {
  def be(o: or.type) = this // or.Type is a singleton type, so it only accepts the above 'or'

  def not(t: to.type) = this //idem

  def be: String = "That is the question"
}

To be or not to be
```

### Named/Default Parameters

Case classes can make a great API which can be very expressive, we can leverage the named parameters of the case classes to write something like this:
`case class DBConnection(url: String, user: String = "postgres", pass: String = "password")`. This is easier to create and probably use than the
fluent DSL approach described before, so it is always a win/win situation.

### Mutable Constructor Pattern

Sometimes having mutable state when used safely, can offer great value, like in the class below which mirrors the behaviour of a fun suite

```scala
class NamedTest(val name: String, val test: () => Unit) // Named test class which represents a test with a name, and the function to execute

abstract class DemoSuite {
  private[this] var _tests = List.empty[NamedTest] // We start the class with an empty list of tests

  protected def test(testName: String)(test: => Unit): Unit = { // Curried parameter list with a by-name parameter, which is transformed into a 0
    // function to avoid issues with lazyness. All we do whenever a new test is register here, it is to add it to the (initially empty) test list.
    // This will be done during construction of a new instance, which is the one time use case that is safe to use mutable state. Nobody else has
    // a reference to the item yet and there is no race contentions here, hence the name mutable constructor pattern
    _tests = new NamedTest(testName, () => test) :: _tests
  }

  protected lazy val tests: List[NamedTest] = _tests.reverse

  def run(): Unit = {
    for (namedTest <- tests) {
      print(s"Running ${namedTest.name}: ")
      try {
        namedTest.test()
        println("Passed")
      }
      catch {
        case NonFatal(ex) =>
          println(s"Failed ${ex.getMessage}")
      }
    }
  }
}
```

## XML, JSON and Extractors

### XML in Scala

XML support now comes in a separate module that needs to be imported into your project `"org.scala-lang.modules" %% "scala-xml" % "1.0.6"`. You  
can import an XML file like `scala.xml.XML.loadFile(theFile)`. In scala you can use XPath using '\' instead of the  '/' operator, like in
`Books \ authors` and you can use the `NodeSeq` result in a for comprehension. In case you want to extract attributes of the Node, use something like
`Books \ "@year"`. Another nice feature is that you can write XML directly in scala: `val xml= <node><child1>{age}</child1></node`, note the `{age}`
which is scala code. You can also match on components of the xml:

```text
book match {
  case <book>{contents @ _*}</book> => ??? // This matches anything within <book>
  case test @ <test>{_ *}</ test > => ??? // we can pattern match the whole thing too
  case _ => _
}
```

### XML serialization

We can serialize a xml element by creating a case class matching the model, and then use xml literals like in:

```scala
case class Person(name: String, age: Int)

def serialize(person: Person) = <Person>
  <name>
    {person.name}
  </name>
  <age>
    {person.age}
  </age>
</Person>
```

Reversing this will look something like `def xmlToPerson(xml:scala.xml.Elem) = Person((xml \ "name").text, (xml \ "age").text.toInt)`.

### JSON in Scala

There is no native support for JSON, but there are many 3rd party libraries for this, we'll use play JSON for the examples.

```scala
import play.api.libs.json._

val fred = Json.parse("""{"name": "fred", "age": 20}""")
fred \ "name" // Similar to the XML approach, we can extract elements like this
(fred \ "age").asOpt[Int] // Unlike the above, this does not throw exceptions
```

To create a Json, we can do something like this:

```scala
val book = JsObject(Map(
  "name" -> JsString(name),
  "age" -> JsNumber(age)
))
// or alternatively
Json.parse(s"""{"name":"${name}", "age":${age}"}""")
```

### Play JSON Printing and Standard Definitions

If we have a JsonValue we can call the _toString_ method on it to display in a compact way, or call `Json.prettyPrint(myJson)` which will display the
json with indents and formatted.

### Custom Serializers

You can suply your own serizalizer by making available an Implicit type class `Format[YourClass]` in the way:

```scala
case class MyType(id: String)

object MyType {
  implicit val mySerializer: Format[MyType] = new Format[MyType] {
    def reads(json: JsValue): JsResult[MyType] = JsSuccess(MyType((json \ "id").as[String]))

    def writes(o: MyType): JsValue = JsObject(Map("id" -> JsString(o.id)))
  }
}
```

You can also use the play macros, like this:

```scala
case class Car(make: String, model: String, year: Int)

object Car {
  implicit val carFormat: Format[Car] = Json.format[Car] // With this approach you can't change the names of the members inside
}
```

### Extractors

Using the unapply method, we can create extractors for custom types. The unapply method takes the type we want to start from, and returns an option
with some arity of tuple type inside. This unapply method allows us to do pattern matching. Collections needs its own extractor method
named `unapplySeq` the method signature is a number of parameters and the last one is a sequence, like in:

```scala
 def unapplySeq(s: String): Option[(String, String, Seq[String])] = Try {
  val split1 = s.split("://")
  val protocol = split1(0)
  val rest = split1(1).split("/").toList
  (protocol, rest.head, rest.tail)
}.toOption
```

Regular expressions can also be used in extractors, there is a built in scala feature to do so. For example:

```scala
val URLString = """^([^:]+.)://([^/]+)/(.+)?$""".r // The '.r' creates a Regex automatically from the String.
val URLString(proto, site, rest) = "https://www.google.com/images" // Note we can use this regex directly to pattern match
```

There is a downside with the regex approach, which is that optional elements of the regex would come as null if they are not present, so you'll have
to deal with them using Options.

## Async with Futures

A future is a lightweight parallelism library, to use them you usually have to bring 3 common imports:

```scala
import scala.concurrent._
import scala.concurrent.duration._
import scala.concurrent.ExecutionContext.Implicits.global
```

Create a future is easy, just do `Future{...}`, ssome of the rules with futures you should follow are:

    * Never block or Await in a future (this includes Thread.sleep()) - unless for testing but nowhere else.
    * Try not to mix ExecutionContexts unless you know what you are doing
    * Futures start executing immediately after they are created (do if you don't want this, make it lazy with a function)
    * Scala futures are not cancellable (but if you run in the context of an actor system, you can terminate that)

Futures passes for several states:

    * Right after creation, the future returns a `Option[scala.util.Try[T]]`, where T is the return type of your future. If the result of that
      option is None, then the future is not yet completed (you can check with the `.isCompleted` function.
    * If the above returns Some([Failure|Success]) then the future is completed

### Composing Futures

Probably the single most useful thing about Futures is that they compose with `map` and `flatMap` while remaining asynchronous. `flatMap` also works
asynchronously. Combining with for expressions is even more powerful. With futures we can line up all the work to be done calling `flatMap` and `map`
which won't block at any point and each completion even would fire the next path, unless it fails somewhere down the line, failures are carried
forward.

If you have an operation returns a future, and you want to apply another function over the returned value (once the future is completed) and that
second function returns another future, instead of having an awkward `Future[Future[...]]` you can use flatMap, but if you have many of those, a more
idiomatic way to use them is a for comprehension:

```scala
val f1 = Future(10)
val f2 = Future(20)
for {
  a <- f1 // Dereference the value in the for block
  b <- f2
} yield a + b // the yield expression works with the values of the futures
```

### Forcing a result

Eventually you will need to get a result from a future. Use `Await.result` or `Await.ready` for this. `Await.ready` waits for the Future to be
resolved one way or the other before continuing, but does not evaluate the result. If you used `Await.result` in this example, an Exception would be
thrown at that point.

### Future operations

#### Collect

When working with frameworks like akka, you might lose the type of the return of the future, to narrow it down you can use `collect`, which takes a
partial function so you can get the type out of it:

```scala
val fi = fa.collect {
  case i: Int => i // This would return a Future[Int]
}
```

As this uses a partial function, it might fail if the type is not defined for the PF.

#### Filter

You can apply filters to futures, when you filter over a future and then try to retrieve the value, if the result of the future was filtered out, then
the future itself will return a Failure

#### Transform

This is to define what to do with a Success future and what to do with a Failure future:

```scala
ffi.transform(i => i * 5, { ex => // The first function would be apply in case the future is successful, the second if the future is a failure
  println(ex.getMessage)
  throw new RuntimeException("it failed to filter", ex)
})
```

#### andThen

This method is to handling event side effects. After the future has resolved, either there is a Success or Failure you can pass a PF and decide on
some kind of side effect to happen (IO, sending another message to an external system...):

```scala
Future(2).andThen { // for event side effects
  case Success(i) if i % 2 == 0 => println(s"it's even") // This still returns the original future of 2 after printing the line
}
```

#### onComplete

Handler that fires when the future is completed and can handle again Success or Failure:

```scala
f7.onComplete {
  case Success(i) => println(s"It worked, and the answer is $i")
  case Failure(ex) => println(s"It failed: ${ex.getMessage}")
}
```

#### foreach and failed

`foreach` is a simple async handler for Success and `failed` is the equivalent for Failure.

### Recovering from Failures

You can create futures in Success or Failure state directly with `Future.successful(...)` and `Future.failed(...)` and these don't require an
`ExecutionContext` imported to run into. If the future is a Failure, you can define a handler which is another function returning a Future, and pass
it to the original future with the method `.fallbackTo(otherFuture)` so the function is executed when the original future fails. This replace a future
with the result of the other, but you can also change the behaviour that you want using `.recover` and passing a PF to handle the failure:

```scala
myFuture.recover {
  case _: IllegalArgumentException => 22
}
myFuture.recoverWith {
  case _: IllegalArgumentException => Future.successful(22)
}
```

`recoverWith` allows pretty much for the same behaviour, but instead you can return kick off another future.

### Dealing with multiple Futures

What if you have several futures and you need to combine them to get a result? You use the sequence function:

```scala
val listOfFutures = List(Future(10), Future(20 * 2), Future(33)).map(i => i * i) // List[Future[Int]]
Future.sequence(listOfFutures) // this returns a Future[List[Int]]
```

What if you want combine a function to map to the futures and then the list of futures to apply that to? We use the traverse function like in
`Future.traverse(listOfFutures)(theFunctionToApplyToThem)`.

#### Other useful methods to combine futures

    * Future.firstCompletedOf: returns the value of the sequence of futures that has completed the operation first
    * Future.foldLeft: First argument is the sequence, then the initial value and then the operation to apply
    * Future.reduceLeft: similar to the foldLeft but without the initial value

### Promises

A promise is the "server" to the "client's" Future, by creating a Promise you create a socket into which you can send something. You can also obtain
the Future from the Promise and hand that to someone else. When you put the value (or failure) into the Promise, the Future is resolved with that
outcome.

```scala
val promise = Promise[Int]
val future = promise.future
future.isCompleted // false
future.value // None
promise.success(10) // fulfill the promise you can also use promise.failure(...) which is called broken promise
future.isCompleted // true
future.value // Some(Success(10))
```

A Promise allows you to easily adapt another asynchronous event into a Scala Future, simply wrap the operation in the promise's future and then you
have the asynchronous behaviour, for example from a Java future to a scala future. This pattern is so common there is already an adaptator for
completeable futures in Java:

```scala
import java.util.concurrent.CompletableFuture
import java.util.function.Supplier

val supp = new Supplier[Int] {
  def get: Int = {
    Thread.sleep(500)
    10
  }
}
val cf = CompletableFuture.supplyAsync(supp) // This returns a java Future
```

But to work with the above, most likely you'll want to do something like:

```scala
import scala.compat.java8.FutureConverters._

val sf2 = cf.toScala // we can implicitly convert this to a scala future and call for Await on it
Await.result(sf2, 1.second) // result is still a Int:10
```

### Future Patterns

#### Batching

Imagine that you have to work with a million files, and then you loop over those and open them to perform some operation. If you are using future you
might exhaust the OS file handlers pool because the files would be opened at the same time. You might want to throttle down the file opening operation
to allow the code to complete whatever it needs to do with the files before opening other files. You can batch the operations and do something like:

```scala
def processSeq(xs: Vector[Int]): Future[Vector[Int]] = {
  val allFutures: Vector[Future[Int]] = xs.map(calc) // calc is a function that takes time to return
  Future.sequence(allFutures) // Returns a Future(List[...])
}

def processSeqBatch(xs: Vector[Int], batchSize: Int): Future[Vector[Int]] = {
  val batches = xs.grouped(batchSize) // we create batches
  val start = Future.successful(Vector.empty[Int]) // This is just the seed, the starting value
  batches.foldLeft(start) { (accF, batch) => // accF is an acumulator of the result
    for {
      acc <- accF // We make sure the accumulator has finished first
      batchRes <- processSeq(batch) // we call the operation that returns a future
    } yield acc ++ batchRes // When the operation has completed then we combine the futures together
  }
}
```

#### Retrying

An asynchronous retrying, when to are talking to external services you don't want to do that without using Futures. You can chain `.fallbackTo`
calls a number of times to retry the operation, but this does not give any flexibility in the number of retries to attempt, or else another way to do
this is:

```scala
def retry[T](op: => T, retries: Int): Future[T] = Future(op) recoverWith { case _ if retries > 0 => retry(op, retries - 1) }
```

We use a byName function and recursion to call the function several times until it exhaust the retries or it has been successful. But this doesn't
give us the ability to back of the operation (i.e. give some time to the server to recover), for that we can use something like:

```scala
object RetryBackoff extends App {

  import akka.actor._
  import akka.pattern.after

  val as = ActorSystem("timer")
  val scheduler: Scheduler = as.scheduler
  val ec: ExecutionContext = as.dispatcher

  def retryBackoff[T](op: => T, backoffs: Seq[FiniteDuration]) // Takes the byName function and a Seq of FiniteDuration defining the wait intervals
                     (implicit s: Scheduler, ec: ExecutionContext): Future[T] =
    Future(op)(ec) recoverWith {
      case _ if backoffs.nonEmpty =>
        // This basically instruct the scheduler to execute the retry after the duration retrieved from the head of the sequence of durations
        after(backoffs.head, scheduler)(retryBackoff(op, backoffs.tail))
    }

  vartime = 0

  def calc(): Int = {
    if (time > 3)
      10
    else {
      time += 1
      println("Not yet!")
      throw new IllegalStateException("not yet")
    }
  }

  val f5 = retryBackoff(calc(), Seq(500.millis, 500.millis, 1.second, 1.second, 2.seconds))(scheduler, ec)
  println(Await.ready(f5, 10.seconds))
  as.terminate() // remember to terminate the actor system or your app will never exit
}
```

# Scala Advanced, Part 3 - Functional Programming, Performance

## FP part 1, Tail Calls, Trampolines, ADTs

If the last call of a function in scala is a self recursion call, then scala will optimize the code with what is known as tail recursion, turning the
recursion into a while loop, preventing stack overflows. For that make sure that the last call is indeed the function, so this won't be tail
recursive:

```scala
def fact2(n: Int): Long = if (n < 2) n else (n * fact2(n - 1)) // the multiplication of fact2 by n make the last call not tail recursive
```

But this is:

```scala
@tailrec // This is not required, but if we make a mistake it won't compile
def fact3(n: Int, acc: Long = 1): Long = if (n < 2) acc else fact3(n - 1, acc * n) // notice the last call doesn't have a factor to multiply with
```

### Mutual calling functions

What if the recursion is between two functions like in:

```scala
object Daft { // The funtions call each other until 0 is reached
  def odd(n: Int): Boolean =
    n match {
      case 0 => false
      case x => even(x - 1)
    }

  def even(n: Int): Boolean =
    n match {
      case 0 => true
      case x => odd(x - 1)
    }
}
```

We can fix this problem by using algebraic data types and a pattern called trampoline. Imagine the following ADT which have a super type and two
subtypes:

```scala
sealed trait Bounce[A]

case class Done[A](result: A) extends Bounce[A]

case class Call[A](nextFunc: () => Bounce[A]) extends Bounce[A]
```

`Call` is a type indicating there is more work to do, so instead of descending into the stack, we are going to use this structure to `Bounce` over the
top of the stack without descending to it. So we can transform the previous even calculation into something like:

```scala
object StillDaft {

  def even(n: Int): Bounce[Boolean] =
    n match {
      case 0 => Done(true)
      case x => Call(() => odd(x - 1))
    }

  def odd(n: Int): Bounce[Boolean] =
    n match {
      case 0 => Done(false)
      case x => Call(() => even(x - 1))
    }
}

@tailrec // This is now tail recursive because the last call in the `Call` branch is `trampoline` itself
def trampoline[A](bounce: Bounce[A]): A =
  bounce match {
    case Call(nextFunc) => trampoline(nextFunc()) // nextFunc() returns a Bounce
    case Done(x) => x
  }

trampoline(StillDaft.even(5))
```

As it can be seen the `odd` and `even` methods returns a lazy function (needs to be evaluated) so trampoline would invoke the function each time it
tries to evaluate the parameter `bounce` passed, making it tail recursive. The important thing to notice here is that the execution is split in two
separate places, the logic is within the `StillDaft` object, but the execution happens on the trampoline. Scala has its own implementation of this
in `scala.util.control.TailCalls`, which usage is like this the code below:

```scala
import scala.util.control.TailCalls._

object StillDaft extends TailRec[Boolean] { // we need to mixin the TailRec trait with the parameter being the return type
  def even(n: Int): TailRec[Boolean] =
    n match {
      case 0 => done(true) // similar to the previous Done
      case x => tailcall(odd(x - 1)) // equivalent to the trampoline calls, the function here is not a 0 parameter function, but a byName parameter
    }

  def odd(n: Int): TailRec[Boolean] =
    n match {
      case 0 => done(false)
      case x => tailcall(even(x - 1))
    }
} // we don't need to wrap this up anymore, the call to the method would be like `StillDaft.even(3).results` and this will handle the recursion.
```

## Functors, Monads and Applicative Functors

A for expression gives you a way to navigate through the success path of your code (always the same type, if the for is over options then your
generators needs to be over options, if it is over a collection then generators are over collections...). Each of the generators is like applying
a `flatMap` except for the last `yield` which is equivalent to a `map` (hence the need to have same container types otherwise we'll violate the method
type signatures). A guard in a generator calls underneath the `withFilter` function.

### Functor

A functor is a type class which has an operation (map) that has to comply with `def map[A, B](fa: F[A])(f: A => B): F[B]`. It needs to satisfy two
mathematical laws:

    * Should implement identity which we can express like x.map(identity) = x
    * Composition x.map(b(a(_))) = x.map(a).map(b)

### Monad

A monad is a bit more complicated, needs to implement `flatMap[T, U](fn: T => M[U])(M[T]): M[U]` and satisfy:

    * Left identity: If we have a function from T to M[T] that we call apply then `apply(x).flatMap(f) = f(x)`
    * Right identity: `apply(x).flatMap(apply) = apply(x)` This basically test that apply is repeteable
    * Associativity: `(apply(x).flatMap(f)).flatMap(g) = apply(x).flatMap(x => f(x).flatMap(g))` This means that it is repeteable whatever order
      we apply the flatMaps on it

The point of the above laws is that having proven the above laws, you can start generalizing about the container itself and raise the abstraction
further to the container that holds the type.

### Applicative functors

Often people call these just applicatives and they don't have built-in syntax support. An applicative combines the results from two containers into a
single container of the same type. i. e. `f(Option[Int], Option[Int]):Option[Int]`. This concept is important because it means that you can apply a
function to multiple things each independently and not dependant on each other, an example of this is:

```scala
Future(op1).zip(Future(op2)) match { // op2 is triggered at the same time than op1, which won't be the case if we put it in a for expression
  case (a, b) => a + b
}
```

Same would be applicable to a function like `zipWith`, given two containers and a function, an applicative functor would yield the results of
combining the inner types of the containers in the function provided:

```scala
  def zipWith[A, B, R](o1: Option[A], o2: Option[B])(fn: (A, B) => R): Option[R] =
  zip(o1, o2) match {
    case Some((a, b)) => Some(fn(a, b))
    case _ => None
  }
```

Some libraries like ScalaZ or Cats have cartesian syntax to deal with this, like in `(op1 |@| op2).map(_ + _)`, but in cats this has been deprecated
in favour for `(op1,op2).mapN(_ + _)`.

### Functor/Monad patterns

#### IO

Used to isolate Input/Output effects in a functional way. Remains purely functional until the effect is run.

```scala
val program = cats.effect.IO {
  // Some operation here that performs reading or calling a WS... The program is fully functional at this point, it has not run
  // Once this code is run it can fail, but we captura
}

program.unsafeRunSync() // At this point the program is no longer functional but we capture the side effects of this op and handle it
```

##### Composing IO

The above can be composed like this:

```scala
val program1 = cats.effect.IO {
  ...
}
val program2 = cats.effect.IO {
  ...
}

val program3 = for {
  p1 <- program1
  p2 <- program2
} yield SomeCaseClass(p1, p2) // This returns an effect, but hasn't been evaluated

program3.unsafeRunSync()
```

#### Reader

Reader is the functional way to do dependency injection. It is used to pass defer function application until dependencies are available, another
option for dependency injection (with caveats...)

```scala
case class User(name: String, sessionId: String)

case class DBConn(name: String) {
  def getUser(id: Int): User = User("Dick", "1234")

  def putUser(user: User): Unit = println(s"Updating $user")
}

object UserList {
  def updateUser(id: Int, newName: String) =
    Reader {
      (db: DBConn) => db.putUser(db.getUser(id).copy(name = newName))
    }
}

val updateU = UserList.updateUser(123, "Ricardo")

updateU.run(DBConn("mydb"))
```

#### Writer

Carries an accumulating writer or logger along to track or log work done, follow provenance, etc.

```scala
import cats.data.Writer

def authorized(name: String) = Writer(List("Verifying user"), name.length >= 3)

def greet(name: String, loggedIn: Boolean) = Writer(List("Greet user by name"), s"Hello ${if (loggedIn) name else "User"}")

import cats.instances.list._

val name = "Dick"

val result: Writer[List[String], String] = for {
  loggedIn <- authorized(name)
  greeting <- greet(name, loggedIn)
} yield greeting

result.run
```

#### State

Carries some world state along that is passed between uses (state without mutability). So when you call this monad you get the result of whatever
execution you want to run, along with the updated state to pass to the next state monad.

```scala
val take = State[Queue[Int], Int] { // The first parameter is the 'state' and the second the return of your computation
  case Queue() => sys.error("queue is empty")
  case q => (q.init, q.last)
}

def put(a: Int) = State[Queue[Int], Unit] { q => (a +: q, ()) }

def useQueue: State[Queue[Int], (Int, Int)] = for {
  _ <- put(3)
  a <- take
  b <- take
} yield (a, b) // note that the Queue is not indicated, but it is passed along

useQueue.run(Queue(5, 8, 2, 1)).value
```

#### Free

As long as you get have a function, you can get the free monad from it. You start by creating an ADT of the basic things you want to do, the free
monad is then used to combine this ADT into effectively a List of stuff to do which is handed to an interpreter (this interpreter can be swapped out
to do different things).

```scala
sealed trait DBFree[A]

case class Save[T](id: Int, value: T) extends DBFree[Unit]

case class Load[T](id: Int) extends DBFree[Option[T]]

case class Remove(id: Int) extends DBFree[Boolean]

case object ShowState extends DBFree[Unit]

type DB[A] = Free[DBFree, A] // this is cat providing this type, Free is the 'entity' that would be responsible to execute the 'steps' passed

import cats.free.Free.liftF // lifters, optional but make life easier. This creates a 'step' that can be composed and executed by the Free monad

def save[T](id: Int, value: T): DB[Unit] = liftF[DBFree, Unit](Save[T](id, value))
def load[T](id: Int): DB[Option[T]] = liftF[DBFree, Option[T]](Load[T](id))
def delete(id: Int): DB[Boolean] = liftF[DBFree, Boolean](Remove(id))
def showState(): DB[Unit] = liftF[DBFree, Unit](ShowState)

// With this in place we can create a program
def modify[T](id: Int, fn: T => T): DB[Unit] =
  for {
    vOpt <- load[T](id)
    updatedOpt = vOpt.map(v => fn(v))
    _ <- updatedOpt
      .map(updated => save[T](id, updated))
      .getOrElse(Free.pure(())) // We need to give back something in case we don't find the item. Pure is like the 'Done' from the
    // trampoline examples, we need to pass a value, in this case is unit
  } yield ()
```

The above can be further composed to construct a program with a high level of complexity, but at this point we haven't run anything, we have the Free
constructed but we need an interpreter to run it:

```scala

def program: DB[Option[Person]] =
  for {
    _ <- showState()
    p <- load[Person](123)
  } yield p

def interpreter: DBFree ~> Id = new (DBFree ~> Id) { // Id is a random type, could be a future for example
  // ~> is a convenient type to generically promote DBFree ~> Id into
  // DBFree[T] ~> Id[T] without some of the mess that comes along with it.  
  val fakeDB = mutable.Map.empty[Int, Any]

  def apply[A](fa: DBFree[A]): Id[A] =
    fa match {
      case Save(id, value) => fakeDB(id) = value
      case Load(id) => fakeDB.get(id).flatMap(x => Try(x.asInstanceOf[A]).toOption)
      case Remove(id) => fakeDB.remove(id).isDefined
      case ShowState => println("State now ---------------- ${fakeDB}")
    }
}

val result: Option[Person] = program.foldMap(interpreter)
```

## Macros

Do not use macros.

### Scala compiler phases

Similar to java, you can compile a class in scala and run it with:

```shell
scalac SomeFile.scala
scala SomeFile
```

You can see the phases of the compilation by passing the argument `-Xshow-phases` to `scalac`. If you see the phases, macros will run between the
_typer_ and the _patmat_ phases. You can dump the AST with `scalac -Xprint:<phase_name>`, you can as well do `scalac -Ybrowse:<phase_name>` to put a
graphic representation of the AST instead (which opens in a new window) although this feature is experimental.

### Macro overview

Some of the macro takes:

    - Macros fire after the typer phase of the compiler
    - Create a method with the signature you want, that calls another method using the `macro` keyword
    - Create the macro method to take the AST and return another AST (maybe altered)
    - Use pattern matching and/or quasiquotes to disassemble, and potentially alter the AST
    - Provide warnings or errors to the compiler along the way

### Demo macro

Imagine that we want to provide a useful toString() method to a lambda function such as `val fn = (x:Int) => x+1`. Behind the scenes this will create
a class with an apply method with that signature, and if we call the `toString` method on it we get something like
`$Lambda$1052/0x0000000800690040@2a334bac`.

We need the following:

```scala
class NamedFunction[-T, +R](val name: String, fn: T => R) extends (T => R) { // Note that we'll need to create this function for different arities
  def apply(x: T): R = fn(x)

  override def toString: String = name
}
```

Let's start out with a macro that takes a function and just returns the same thing first.

```scala
import scala.language.experimental.macros

object SimpleMacro {
  def describeFn(expr: Int => Int): Int => Int = macro namedImpl // Macro is a keyword in scala, and namedImpl the macro method name

  def namedImpl(c: scala.reflect.macros.blackbox.Context) // This is like the runtime universe
               (expr: c.Expr[Int => Int]): c.Expr[Int => Int] = expr // c.Expr is actually the AST and [Int => Int] the type of it
}
```

We can also do the above for generic types:

```scala
object SimpleMacro {
  def describeFn[T, R](expr: T => R): T => R = macro namedImpl[T, R]

  def namedImpl[T, R](c: blackbox.Context)(expr: c.Expr[T => R]): c.Expr[T => R] = expr
}
```

The above doesn't really do anything, we can start modifying the AST. A way to do this is by using quasiquotes. For this you can do the following:

```scala
import scala.reflect.runtime.universe._

val fbTree = q"(x:Int) => x+1" // this would be passed to the parser and represent the AST as a universe.Function
showCode(fbTree) // This desugars the function to represent it without the implicit transformations

val aVal = q"val a = 10" // This would create an universe.ValDef

// given the above, we can do pattern match on the quasiquotes to extract the meaningful values:
aVal match {
  case q"val x = $v" => s"The value is $v" // bare in mind that the string has to be an exact match
} // res1: String = The value is 10
```

Quasiquotes can get really complex, like:

```scala
import scala.reflect.runtime.universe._

class Animal

trait HasLegs

trait Reptile

val cd =
  q"""class Frog extends Animal with HasLegs with Reptile {
      val color: String = "green"
  }"""

val q"class $c extends $s with ..$t {..$b}" = cd // .. Denotes multiple occurrences, in this case of traits and whatever goes in the body
```

So going back to the example, we could override the toSTring method with something like:

```scala
class NamedFunction[-T, +R](val name: String, fn: T => R) extends (T => R) {
  def apply(x: T): R = fn(x)

  override def toString: String = name
}

object SimpleMacro {
  def describeFn[T, R](expr: T => R): T => R = macro namedImpl[T, R]

  def namedImpl[T, R](c: blackbox.Context)(expr: c.Expr[T => R]): c.Expr[T => R] = {
    import c.universe
    val name = expr.tree match {
      case q"$fn" => fn.toString // the q"$fn" gives us the whole expression of the function passed like invoking the showCode seen before
    }
    // We can also generate compiler warnings, c.enclosingPosition would mark the line where the warning is. Change by c.error or c.abort to fail
    // compilation
    if (name.length > 100) c.warning(c.enclosingPosition, s"Function length exceeds recommended limit for auto-describe")


    c.Expr[NamedFunction[T, R]](q"new NamedFunction($name, $expr)")
  }
}
```

Some macros limitations:

    - Since the macro fires after the `typer` phase, the scala code must type-check before a macro can be used, i.e. the scala syntax must be valid
      before the macro is applied, the macro cannot "fix" invalid syntax
    - Desugaring will already have occurred by the time you get the AST, e.g. our `((x: Int) => x.+(1))` instead of `((x: Int) => x + 1)`
    - Macros can only be used in method positions, i.e. there must be a method call around the expression to be passed to the macro
    - While a blackbox macro can specialize the return type (as in our example), it must follow the type signature rules. Whitebox macros relax this
      restriction a little, allowing a macro to potentially return different types, but at the expense of understandability and also usually tooling
      problems/confusion, IDEs have trouble following the types

## Parser Combinators

Internal DSLs must work within the confines of the Scala language, and type-check. Scala is flexible in this regard, but you can't do something like
`10 PRINT "Hello, World"`. For that, you need to construct an external DSL:

```scala
// Add libraryDependencies += "org.scala-lang.modules" %% "scala-parser-combinators" % "2.1.1"
object BasicParser extends JavaTokenParsers {
  def gotos: Parser[String] = "PRINT" // The parser is parameterized by the type is going to produce

  def fors: Parser[String] = "FOR"

  def tos: Parser[String] = "TO"

  def bys: Parser[String] = "BY"

  def nexts: Parser[String] = "NEXT"

  def prints: Parser[String] = "PRINT"
}
```

`JavaTokenParsers` extends `RegexParsers`, so it provides support for whitespace matching and handling, String and Regex implicits, ident,
wholeNumber, decimalNumber, floatingPointLiteral, stringLiteral... In the case we want to create a parser for int, we can do the following:
`def int: Parser[Int] = wholeNumber ^^ {_.toInt}`. Where `^^` transforms from the parsed type (String) to the desired one (Int) using the function
after the `^^`. You can increase the complexity of the parsers, for example to create case classes:

```scala
case class LineNumber(line: Int)

def lineNumber: Parser[LineNumber] = """\d+""".r ^^ { ln => LineNumber(ln.toInt) }
```

This is the building block of more complex parsers, as you can start to compose parsers within parsers, for example given the ADT:

```scala
sealed trait StatementLine

case class Print(printList: Seq[String]) extends StatementLine

case class For(variable: Variable, start: Int, end: Int, by: Option[Int]) extends StatementLine

case class Goto(lineNumber: LineNumber) extends StatementLine

case object Next extends StatementLine

case class CompleteLine(line: LineNumber, statement: StatementLine)
```

You can create parsers to support the above:

```scala
// (param ignored) nexts is the method of BasicParser defined before which contains the token identifier
def nextStatement: Parser[Next.type] = nexts ^^ { _ => Next }

def gotoStatement: Parser[Goto] =
  gotos ~ lineNumber ^^ { // The ~ is 'and then', is used to connect parsers
    case g ~ ln => Goto(ln)
  }
// alternatively we can do
def gotoStatement: Parser[Goto] = gotos ~> lineNumber ^^ Goto // ~> and <~ match but throw away tokens to left and right, respectively
```

To get this running, you'll need to create something like:

```scala
def completeLine: Parser[CompleteLine] = lineNumber ~ (printStatement | gotoStatement | nextStatement | forStatement) ^^ {
  case lineNumber ~ statement => CompleteLine(lineNumber, statement)
}
// passing the method that parses the whole line and the line itself
def apply(input: String): CompleteLine = parseAll(completeLine, input) match {
  case Success(result, _) => result
  case failure: NoSuccess => scala.sys.error(failure.msg)
}
```

For certain parser specially "left recursive" parsers (scala combinators can't easily backtrack the parsers) , you need to use a special type of
parsers named packrat parsers. To use them mix in the `PackratParsers` trait so `:Parser[T]` becomes `:PackratParser[T]`. Then replace `def` with
`lazy val` like this:

```scala
object BasicParser extends JavaTokenParsers with PackratParsers {
  //…
  lazy val completeLine: PackratParser[CompleteLine] =
    lineNumber ~
      (printStatement | gotoStatement | nextStatement | forStatement) ^^ {
      case lineNumber ~ statement => CompleteLine(lineNumber, statement)
    }
```

## Performance and Optimization

More than 95% of your potential optimization is in less than 5% of your code. Find the hotspots, prove the problem, and fix it. Benchmark your
code as a first measure (time it), but be aware that hotspot results can vary, never base the decision about an optimization on a single run
through the code, make a few separate runs to verify and watch all the results.

The correct use of collections is one of the things that can yield great improvements in performance, for example appending to a list is very slow
(list are immbutable so each time you add an element to it, it needs to traverse and copy the whole list). As an alternative you can use a Vector,
which has a constant time for appends. Also, head appends (adding an item at the head of the list) is much faster that appending it to the end. So
you can add it all the time to the head of the list, and then reverse the collection. Also, vectors are better for larger collections than lists.
Also be mindful about the boxing/unboxing types, which adds computation time. Arrays are also more performant than other collections, but they are
unsafe so use them with caution.

Operations in CPU registers are faster than library calls, for example `x * x` is faster than `Math.pow(x,2)`. In the same line, bitwise
operations are sometimes like calling an operation (multiplying by 2 is the same than bitsifting the number to the left 1 spot). For example in
hashing functions is often performed a multiplication by 31, which most compilers will do as a bitshift 5 minus 1. On the same line, use tail
recursion whenever possible due to the while loop optimization.

### Profiling the code

Oracle shifts with [VisualVM](https://visualvm.github.io/download.html) which is distributed with the openJDK as well. Before you can use it, you
need to go through the following:

    * Establish a test or main method that runs longer than a few seconds over the code to examine
    * Start visualvm on your computer (and do any setup/performance calibration)
    * Run your test or main method, and attach visualvm when it shows up in the left pane
    * Go to the Sampler tab when ready, and start CPU sampling
    * When ready (after some data has been gathered) take a snapshot and examine it
    * Drill down into the hotspots
    * Refactor code in hotspots, and repeat
    * Stop when the performance is acceptable (spend your effort where it's needed)

### Caching

Caching increase performance by not having to run a calculation, but it comes as a trade off with memory. One way to achieve caching is by the use
of lazy val's, which would hold the value of a computation after it runs.
