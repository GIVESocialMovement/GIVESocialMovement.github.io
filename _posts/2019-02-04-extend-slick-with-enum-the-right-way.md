---
layout: post
title: "Use sealed type as Enum and make it work with Slick"
date: 2019-02-04
author:
  name: "@tanin"
  url: https://twitter.com/tanin
---

Since the inception of our Playframework system in Scala, we used `scala.Enumeration` with [Slick](https://github.com/slick/slick). We translate an enum to string (using `toString`) when storing it in the database. Here was how we did it:

```
import slick.jdbc.PostgresProfile.api._

object Country extends Enumeration {
  val Singapore, Thailand = Value
}

object CountryType {
  def countryToSqlType(country: Country.Value) = country.toString
  implicit val CountryColumnType = MappedColumnType.base[Country.Value, String](countryToSqlType, Country.withName)
}

case class Campaign(
  ...
  country: Country.Value
  ...
)

class CampaignTable(tag: Tag) extends Table[Campaign](tag, "campaigns") {

  import CountryType._

  def country = column[Country.Value]("country")

  def * = (
    ...
    country,
    ...
  ) <> ((Campaign.apply _).tupled, Campaign.unapply)
}
```

There are two issues with the code above:

1. `scala.Enumeration` doesn't fail compilation when a pattern matching on it isn't exhaustive. I've heard that Scala 3 has fixed this counter-intuitive behaviour.
2. Whenever we touch `Country.Value`, we need to bring the implicit `CountryColumnType` into scope.

Number 1 is a huge coding trap, and Number 2 is a mild verbosity problem.

In this blog, I want to propose a better way to solve each of these two issues.


## Use sealed type instead of Enumeration

A `sealed` type triggers compilation failure when a pattern matching on it isn't exhaustive. One added benefit is that it is also more powerful as its choices can have their own methods and members.

We want to utilize a `sealed` type while maintaining the `withName` and `toString` method of `scala.Enumeration`.

And, obviously, with Scala's super power, we can achieve that in a concise way. Here's how we can achieve it:

```
object Country {
  sealed abstract class Value {
    lazy val name: String = {
      // Note that we cannot use getSimpleName/getCanonicalName because it would raise "Malformed class name".
      val n = getClass.getName.stripSuffix("$")
      n.split("\\.").last.split("\\$").last
    }

    override def toString: String = name
  }
  object Thailand extends Value
  object Singapore extends Value

  def withName(s: String): Value = {
    import scala.reflect.runtime.universe._

    val symbol = typeOf[Value].typeSymbol.asClass.knownDirectSubclasses.find(_.name.decodedName.toString == s).get
    val module = reflect.runtime.currentMirror.staticModule(symbol.fullName)

    reflect.runtime.currentMirror.reflectModule(module).instance.asInstanceOf[Value]
  }
}
```


## Extend PostgresProfile instead of importing the ad-hoc implicit conversion

We can simply extends `slick.jdbc.PostgresProfile` and override the `api` member as shown below:

```
trait OurExtendedPostgresProfile extends slick.jdbc.PostgresProfile {
  class API extends super.API {
    implicit val countryMapper: BaseColumnType[Country.Value] = MappedJdbcType.base[Country.Value, String](_.toString, Country.withName)
  }

  override val api: API = new API
}

// Please note that we need to make a trait first. Otherwise, we would encounter an AbstractMethodError.
// See why: https://github.com/tminglei/slick-pg/issues/367
object OurExtendedPostgresProfile extends OurExtendedPostgresProfile
```

Then, in places where we import `slick.jdbc.PostgresProfile.api._`, we instead import `OurExtendedPostgresProfile.api._`. And that's it.


## Make Slick work with all Enums

As we all know, Scala system is powerful and safe at the same time. It should be possible to make Slick work with all  defined Enums. Here's how we do it:

First, we define base classes for all Enums:

```
package framework

import scala.reflect.runtime.universe._

object Enum {
  def withName[T <: Enum#EnumValue](s: String)(implicit tt: TypeTag[T]): T = {
    val symbol = typeOf[T].typeSymbol.asClass.knownDirectSubclasses.find(_.name.decodedName.toString == s).get
    val module = reflect.runtime.currentMirror.staticModule(symbol.fullName)
    reflect.runtime.currentMirror.reflectModule(module).instance.asInstanceOf[T]
  }
}

class Enum {
  type Value <: EnumValue

  protected[this] abstract class EnumValue {
    lazy val name: String = {
      // Note that we cannot use getSimpleName/getCanonicalName because it would raise "Malformed class name".
      val n = getClass.getName.stripSuffix("$")
      n.split("\\.").last.split("\\$").last
    }

    override def toString: String = name
  }

  def withName(s: String)(implicit tt: TypeTag[Value]): Value = Enum.withName[Value](s)
}
```

With the defined base classes, the `Country` enum becomes:

```
object Country extends framework.Enum {
  sealed abstract class Value extends EnumValue
  object Thailand extends Value
  object Singapore extends Value
}
```

Then, we can make a `PostgresProfile` that works with all subclasses of `EnumValue` as shown below:

```
object OurExtendedPostgresProfile extends slick.jdbc.PostgresProfile {

  class API extends super.API {

    implicit def baseEnumMapper[T <: framework.Enum#EnumValue](
      implicit tt: reflect.runtime.universe.TypeTag[T],
      clazz: ClassTag[T]
    ): BaseColumnType[T] = {
      MappedJdbcType.base[T, String](tmap = _.toString(), tcomap = Enum.withName(_)(tt))
    }

    // See why we need the below here: https://github.com/slick/slick/issues/1986
    implicit def getOptionMapper2TT[B1, B2 <: framework.Enum#Value : BaseTypedType, P2 <: B2, BR] = OptionMapper2.plain.asInstanceOf[OptionMapper2[B1, B2, BR, B1, P2, BR]]
    implicit def getOptionMapper2TO[B1, B2 <: framework.Enum#Value : BaseTypedType, P2 <: B2, BR] = OptionMapper2.option.asInstanceOf[OptionMapper2[B1, B2, BR, B1, Option[P2], Option[BR]]]
    implicit def getOptionMapper2OT[B1, B2 <: framework.Enum#Value : BaseTypedType, P2 <: B2, BR] = OptionMapper2.option.asInstanceOf[OptionMapper2[B1, B2, BR, Option[B1], P2,Option[BR]]]
    implicit def getOptionMapper2OO[B1, B2 <: framework.Enum#Value : BaseTypedType, P2 <: B2, BR] = OptionMapper2.option.asInstanceOf[OptionMapper2[B1, B2, BR, Option[B1], Option[P2], Option[BR]]]
  }

  override val api: API = new API
}
```

Special thank to @hvesalai for pointing out what we need ([https://github.com/slick/slick/issues/1986](https://github.com/slick/slick/issues/1986)).

Now you can easily add a new Slick-compatible Enum with minimal code.

PS. We're always looking for an improvement that makes the code more concise and elegant. If you have an idea, please open an issue in our github repo to start a discussion. Thank you!
