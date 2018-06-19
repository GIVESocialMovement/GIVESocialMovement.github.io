---
layout: post
title: "Compilation safety on Playframework's i18n with Scala Macros"
date: 2018-06-17 20:24:00 -08:00
author:
  name: "@tanin"
  url: https://twitter.com/tanin
---

[Playframework's i18n mechanism](https://www.playframework.com/documentation/2.6.x/ScalaI18N) isn't very safe. Well, it's not as safe as I'd like  as a person who prefers Scala because of its safeness. A translation is defined in `messages`, for example, `page.index.hello = Hello {0}`. Then, somewhere in a template, we invoke `@request.messages("page.index.helllo", "Tanin")`.

With the above way of doing things, we can easily make a few mistakes. We could misspell the key. We could forget to invoke the method with an appropriate number of arguments. To make the matter worse, Playframework will compile it successfully. You will only see the mistake(s) when you render the page. Your message will look like `page.index.helllo` (did you notice the three `l`s?) because the misspelled key isn't defined. Avoiding mistakes when there are thousands of translations in a project is painful.

I made the exact mistake, and I've realised afterwards that there is a way to catch this kind of mistakes with Scala Macros!

<!---excerpt--->

Scala Macros is a dark powerful magic. The knowledge and wisdom are scattered around internet. There are blogs and Stackoverflow's answers here and there. Some are outdated. Then, there's Scala Meta, which is like a fork of Scala Macros. Only a handful of Scala programmers (like [Eugene Burmako](https://twitter.com/xeno_by) who answered a lot of Stackoverflow's questions around Scala Macros) seem to hold the source of truth.

What I understand about it is that Scala Macros is the code that manipulates abstract syntax trees of our codebase. It's like I manually replace the method invocation of `test(..)` with `anotherTest(..)` right before `sbt compile` except that I write Scala to do that for me.

One way to add compilation safety to the Playframework's i18n mechanism is to use Scala Macros to read the method's arguments and verify them against the translation within `messages`. The Macros code is actually pretty short. It looks like below:

```scala
 package givers.translation

 import java.io.File
 import java.text.MessageFormat

 import play.api.i18n.Messages

 import scala.reflect.macros.whitebox

 object Translation {
   val messages = {
     val file = new File("conf/messages")
     Messages.parse(Messages.UrlMessageSource(file.toURI.toURL), file.getCanonicalPath).fold(throw _, { m => m })
   }

   def applyImpl(c: whitebox.Context)(key :c.Expr[String], args: c.Expr[Any]*): c.Expr[String] = {
     import c.universe._

     key.tree match {
       case Literal(Constant(k: String)) =>
         messages.get(k) match {
           case Some(text) =>
             val requiredArgSize = new MessageFormat(text).getFormats.length
             if (args.size != requiredArgSize) {
               throw new Exception(s"The key '$k' requires $requiredArgSize arguments. But ${args.size} arguments was given.")
             }
           case None => throw new Exception(s"The key '$k' isn't defined in conf/locale/messages")
         }
       case _ => // do nothing because the key isn't a literal string.
     }


     import c.universe._
     c.Expr[String](
       q"""
         import ${c.prefix}._
         _root_.libraries.TranslationHelper.doNotUseThisMethodDirectlyTranslate($key, ..$args)
        """
     )
   }
 }
```

There are a few things to notice in the code above:

* The code above raises an `Exception` at compilation if, for example, the key is missing.
* We cannot verify arguments in the case where the key isn't a literal string. For this case, we merely replace the method invocation.
* We verify that the key exists and that the number of arguments are correct
* `import ${c.prefix}._` imports the implicit variables on the original scope.

Setting up Playframework to work correctly with Macros is a little bit challenging. So, we've provided a full working example [here](https://github.com/GIVESocialMovement/i18n-compilation-safety-with-macro). Try using translation with invalid key or invalid arguments and see it fail the compilation.

I hope this trick saves you some headache when developing with Playframework's i18n!
