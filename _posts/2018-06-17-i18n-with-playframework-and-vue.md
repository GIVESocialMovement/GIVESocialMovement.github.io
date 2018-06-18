---
layout: post
title: "i18n with Playframework and Vue"
date: 2018-06-17 20:24:00 -08:00
author:
  name: "@tanin"
  url: https://twitter.com/tanin
---

Playframework has its own internationalization mechanism, which use `conf/messages` for storing translations. The mechanism is great for server-side rendering. On Vue's side, there are multiple libraries for internationalization in Vue.js, and it seems [vue-i18n](https://github.com/kazupon/vue-i18n) is the winner. 

Now the question is: How do we use it with Vue.js?

It would be great if (1) [vue-i18n](https://github.com/kazupon/vue-i18n) uses the Playframework's `conf/messages` and (2) the internationalization works well with local development (e.g. hot-reloading).

<!---excerpt--->

Like I mentioned earlier, Playframework already has a mechanism around i18n. For example, we can set locale, which, I think, stores locale as a cookie. We can make a route `/locale/de` and change the user's locale with `Redirect(..).withLang(Locale("de"))`. And we can get the locale if our controller extends from `MessagesAbstractController(..) with I18nSupport`. You can see an example [here](https://github.com/GIVESocialMovement/sbt-i18n/blob/master/test-play-project/app/controllers/HomeController.scala#L21).

One way to get the data from `conf/messages` to [vue-i18n](https://github.com/kazupon/vue-i18n) is, of course, using an [sbt-web](https://github.com/sbt/sbt-web)'s plugin. We can essentially compile `conf/messages` into `messages.en.js` and inject `messages.en.js` into `*.scala.html`. Then, [vue-i18n](https://github.com/kazupon/vue-i18n) can use the translations in `messages.en.js`. [sbt-web](https://github.com/sbt/sbt-web) also offers a mechanism to hot-reload this file. A code in your project may look like below:

```
@(messages: play.api.i18n.Messages)

<script src="https://cdnjs.cloudflare.com/ajax/libs/vue/2.5.16/vue.js"></script>
<script src="https://unpkg.com/vue-i18n/dist/vue-i18n.js"></script>
<script src="@routes.Assets.versioned("locale/messages." + messages.lang.code + ".js")" type="text/javascript"></script>

<p>From server side: <strong>@messages("home.messageFromServerSide")</strong></p>

<p id="app">From client side: <strong>{{ '{{'}} $t('home.messageFromClientSide') }}</strong></p>

<script>
  var app = new Vue({el: '#app'})
</script>
```

We've built [sbt-i18n](https://github.com/GIVESocialMovement/sbt-i18n) to do exactly this. There are a few things to point out in the above code:

* `messages: play.api.i18n.Messages` can be obtained from a request if your controller extends `MessagesAbstractController` and `I18nSupport`
* [vue-i18n](https://github.com/kazupon/vue-i18n) requires that we inject `i18n` when instantiating `Vue`. But that is verbose. As seen in the code above, we didn't do that. Because [sbt-i18n's VueI18nSerializer](https://github.com/GIVESocialMovement/sbt-i18n/blob/master/src/main/scala/givers/i18n/VueI18nSerializer.scala#L20) injects this automatically to every Vue instantiation.

If you are using Playframework/Vue.js and looking for a way to internationalization, try [sbt-i18n](https://github.com/GIVESocialMovement/sbt-i18n).

[sbt-i18n](https://github.com/GIVESocialMovement/sbt-i18n) can be extended to support other JS internationalization as well. We just need to write a new serializer. So, if you are using other JS frameworks and are interested, let us know. We might be able to extend the plugin together.
