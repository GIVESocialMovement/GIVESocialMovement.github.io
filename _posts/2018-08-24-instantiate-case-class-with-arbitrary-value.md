---
layout: post
title: "Instantiate case class with arbitrary values to reduce verbosity in tests"
date: 2018-08-24
author:
  name: "@tanin"
  url: https://twitter.com/tanin
---

As we are using Playframework with Slick, we have 20+ case classes that directly correspond to tables in Postgresql.
We have 10+ case classes that are used for [Playframework's forms](https://www.playframework.com/documentation/2.6.x/ScalaForms) and for higher-level representations (e.g. encapsulating two case classes within a single case class). With a large number of case classes, writing tests, specifically faking these case classes, becomes tedious and verbose.

More than often, I don't care much what the values are. I just want sensible values (e.g. not empty, positive values, not too long). I've asked around a few times on [Reddit](https://www.reddit.com/r/scala/comments/8kaguf/use_scalacheckshapeless_to_instantiate_a_case/) and on [Stackoverflow](https://stackoverflow.com/questions/47482542/generate-a-function-to-instantiate-a-case-class-with-default-values-using-scala). One good suggestion is to use [scalacheck-shapeless](https://github.com/alexarchambault/scalacheck-shapeless).

Scalacheck-shapeless works but isn't a good fit. The main problem is that scalacheck-shapeless generates an extremely wild value. For example, a generated string might be empty or use Japanese characters. A generated integer might be a large negative number. Because our tests aim to verify normal use cases of our application. We don't want to perform some sort of fuzzing tests here. We ended up having to define some values in our case class instances (using `.copy`) to avoid flaky tests. So, the code was still a bit verbose.

Moreover, Scalacheck-shapeless is a little bit too verbose than my liking, especially when we have a special type, say, `com.twitter.util.Time`, we need to bring into scope an implicit Arbitrary for `com.twitter.util.Time`. For example:

```
// import statements are omitted.

case class User(name: String, createdAt: com.twitter.util.Time)

implicit val implicitTime = Arbitrary.apply(Gen.const(Time.now))

val user = implicitly[Arbitrary[User]].arbitrary.sample.get
```

Note that you might think we can encapsulate these lines in a method. We can't because scalacheck-shapeless uses Macros expansion on the code here. Besides, using Macros comes with its own problem which will be explained in the next paragraph. 

Then, I turned to Macros for generating a case class instance with arbitrary values. However, the main problem with using Macros is that our compile time (in `sbt test:compile`) takes 2x longer (from 2-3 minutes to 6 minutes). Ouch. Besides, Macros, whose advantage is to provide compilation safety, isn't really needed here because the generation is only used in tests. We would have caught an error fairly quickly anyway if there is one.

Eventually, I've decided to go with the good old Scala/Java reflection for instantiating case classes with arbitrary values. Here's what the code looks like:

```
object Maker {
  val mirror = runtimeMirror(getClass.getClassLoader)

  var makerRunNumber = 1

  def apply[T: TypeTag]: T = {
    val method = typeOf[T].companion.decl(TermName("apply")).asMethod
    val params = method.paramLists.head
    val args = params.map { param =>
      makerRunNumber += 1
      param.info match {
        case t if t <:< typeOf[Enumeration#Value] => chooseEnumValue(convert(t).asInstanceOf[TypeTag[_ <: Enumeration]])
        case t if t =:= typeOf[Int] => makerRunNumber
        case t if t =:= typeOf[Long] => makerRunNumber
        case t if t =:= typeOf[Date] => new Date(Time.now.inMillis)
        case t if t <:< typeOf[Option[_]] => None
        case t if t =:= typeOf[String] && param.name.decodedName.toString.toLowerCase.contains("email") => s"random-$arbitrary@give.asia"
        case t if t =:= typeOf[String] => s"arbitrary-$makerRunNumber"
        case t if t =:= typeOf[Boolean] => false
        case t if t <:< typeOf[Seq[_]] => List.empty
        case t if t <:< typeOf[Map[_, _]] => Map.empty
        // Add more special cases here.
        case t if isCaseClass(t) => apply(convert(t))
        case t => throw new Exception(s"Maker doesn't support generating $t")
      }
    }

    val obj = mirror.reflectModule(typeOf[T].typeSymbol.companion.asModule).instance
    mirror.reflect(obj).reflectMethod(method)(args:_*).asInstanceOf[T]
  }

  def chooseEnumValue[E <: Enumeration: TypeTag]: E#Value = {
    val parentType = typeOf[E].asInstanceOf[TypeRef].pre
    val valuesMethod = parentType.baseType(typeOf[Enumeration].typeSymbol).decl(TermName("values")).asMethod
    val obj = mirror.reflectModule(parentType.termSymbol.asModule).instance

    mirror.reflect(obj).reflectMethod(valuesMethod)().asInstanceOf[E#ValueSet].head
  }

  def convert(tpe: Type): TypeTag[_] = {
    TypeTag.apply(
      runtimeMirror(getClass.getClassLoader),
      new TypeCreator {
        override def apply[U <: Universe with Singleton](m: api.Mirror[U]) = {
          tpe.asInstanceOf[U # Type]
        }
      }
    )
  }

  def isCaseClass(t: Type) = {
    t.companion.decls.exists(_.name.decodedName.toString == "apply") &&
      t.decls.exists(_.name.decodedName.toString == "copy")
  }
}
```

And here's how we can use it:

```
val user = Maker[User]
val user2 = Maker[User].copy(email = "someemail@email.com")
```

Using reflection is pretty satisfactory because the compile time is still fast and the API is concise (more concise than using scalacheck-shapeless). Here are some other advantages of the code above:

* You can override the values you care about using the `.copy` method of a case class.
* You can customise the generated values in any way you'd like. For example, if the field has the word `email` in it, then we generate a valid random email.
* All values are uniquely generated. This is good because using duplicated values might not catch a certain kind of bugs (e.g. passing wrong values to wrong parameters).
* It works with Enum. It'll always choose the first enum value.

I hope this code will be as useful for your codebase as well.

