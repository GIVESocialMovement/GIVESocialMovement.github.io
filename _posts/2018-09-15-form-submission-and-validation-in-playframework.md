---
layout: post
title: "Form submission and validation in Playframework with any Javascript framework"
date: 2018-09-15
author:
  name: "@tanin"
  url: https://twitter.com/tanin
---

Since we've migrated [GIVE.asia](https://give.asia) from Ruby on Rails to Playframework about a year ago, we have discovered a few of new techniques around Playframework. In this blog, we would like to share how we build form submission and validation that is cleanly integrated with any Javascript framework. This blog serves as a tutorial as well.

The article, [Handling form submission](https://www.playframework.com/documentation/2.6.x/ScalaForms) in the official documentation, focuses on the traditional way of form submission, which revolves around `<form action="post">`. The traditional way isn't suitable when using a Javascript framework. A more suitable way  is to send JSON-encoded data using AJAX.

There are 2 important points that we should keep in mind when building a form submission and validation:

1. A clean way to set default values when rendering inputs to users.
2. A clean way to transform between end-user representation and internal representation. For example, when users input an amount (of dollars), what users interact with is `49.99` (two-decimal floating point), but what we use internally is `4999L` (cents), whose type is `Long`. We want a clean mechanism to transform these 2 values back and forth.

Let's start...

First, let's define our case class to represent the internal data:

```
case class Product(itemName: String, price: Long)
```

Our Play's form is defined as shown below:

```
import play.api.data.{Form, FormError, Forms}
import play.api.data.format.Formatter
import play.api.data.Forms._

val amount = Forms.of(new Formatter[Long] {
  def bind(key: String, data: Map[String, String]): Either[scala.Seq[FormError], Long] = {
     data.getOrElse(key, "") match {
       case "^([0-9]+).([0-9][0-9])$".r(whole, decimals) => Right(whole.toLong * 100 + decimals.toLong)
       case _ => Left(Seq(FormError(key, "error.number")))
     }
  }

  def unbind(key: String, cents: Long) = Map(key -> s"${cents / 100}.${(cents % 100).formatted("%02d")}")
})

val PRODUCT_FORM = Form(
  mapping(
    "name" -> text,
    "price" -> amount
  )(Product.apply)(Product.unapply)
)
```

From the code above, we define the `amount` mapping that converts, for example, `49.99` to `4999L` and vice versa. This is great because we can convert our internal representation (which is Long) back to the end-user representation.

We can draw a line in our controller which converts between our internal representation (case class) and the end-user representation (JSON). Here is a snippet in the controller:

```
def showForm = Action { implicit request =>
  Ok(views.html.form(
    params = Json.toJson(PRODUCT_FORM.fill(Product("iphone", 59999L)).data)
  ))
}

def submit = Action(parse.json) { implicit request =>
  form.bindFromRequest().fold(
    hasErrors = { ... } // handle errors
    success = { data =>
      // data is an instance of Product.
    }
  )
}
```

As you can see above, the controller logic only operates on the case class `Product`.

Now let's look at the view side. Let's assume we use Vue. Our view looks like below:

```
@(params: play.api.libs.json.JsValue)

<html>
  <body>
    <div id="app">
      <input type="text" v-model="params.name">
      <input type="text" v-model="params.price">
      <button v-on:click="submit()">Submit</button>
    </div>

    <script>
      var app = new Vue({
        el: '#app',
        data: {
          params: @Html(params.toString) // in practice, we should encode JSON in Base64 to avoid cross-site scripting.
        },
        methods: {
          submit: function() {
            // We simply submit `this.params` using AJAX to the controller.
          }
        }
      })
    </script>
  </body>
</html>
```

In Vue, we don't do anything with `params`. We simply use `params` in textboxes. Notice that the default values are set from our controller. For example, `59999L` is converted to `599.99` when we invoke `PRODUCT_FORM.fill(Product("iphone", 59999L)).data`, and `599.99` is what users see.

We like this approach because there is a clear boundary. Our view operates on JSON; Our controller immediately converts JSON to `Product` and only operates on `Product`. The validation and transformation logic is defined in one place, which is in `PRODUCT_FORM`.

We had been happy with this approach and `play.api.data.Form` ... until we started using a `Seq` in our form like below:

```
case class ComplexProduct(itemName: String, price: Long, images: Seq[String])

val COMPLEX_PRODUCT_FORM = Form(
  mapping(
    "name" -> text,
    "price" -> amount,
    "images" -> seq(text)
  )(ComplexProduct.apply)(ComplexProduct.unapply)
)
```

A `Seq` in Play's form doesn't work well with JSON. For example, with the explained approach above, `ComplexOrder("iphone", 59999L, Seq("iphone.png", "iphone2.png"))` would be converted to:

```
{
  "name": "iphone",
  "price": "599.99",
  "images[0]": "iphone.png",
  "images[1]": "iphone2.png"
}
```

And our Vue code can't operate on this kind of array encoding. It's also tricky and hacky to work with this kind of array encoding. This problem has led us to build [our own Play's form library that is fully compatible with JSON](https://github.com/GIVESocialMovement/play-json-form).

Well, I hope this is helpful for anyone looking to implement form submission and validation on Playframework with a Javascript framework.

The working example can be founded [here](https://github.com/GIVESocialMovement/play-json-form/example-project). Though our [play-json-form](https://github.com/GIVESocialMovement/play-json-form) is used in the example, the architecture follows what is explained in this blog.

If you have any question or would like to discuss about anything, please feel free to open a Github issue [here](https://github.com/GIVESocialMovement/GIVESocialMovement.github.io/issues). We are always looking for ways to improve.
