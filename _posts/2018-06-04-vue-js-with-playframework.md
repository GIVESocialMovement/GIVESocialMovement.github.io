---
layout: post
title: "Vue.js with Playframework"
date: 2018-06-04 16:24:00 -08:00
author:
  name: "@tanin"
  url: https://twitter.com/tanin
---

We've finally extracted out our Vue.js plugin for Playframework, <a href="https://github.com/GIVESocialMovement/sbt-vuefy">sbt-vuefy</a>, and shared it with the world.

It has been quite an effort to make Vue.js work seamlessly well with Playframework. This is especially true when we want to avoid using <a href="https://en.wikipedia.org/wiki/Single-page_application">Single-page application (SPA)</a>.

<!---excerpt--->

At <a href="https://give.asia">GIVE.asia</a>, we've decided to go with Server-side rendering, the traditional way, as opposed to Client-side rendering or SPA. Because our site isn't highly interactive, and, with SPA, some website's fundamentals (e.g. SEO, Routing) become complex
and require a lot more Javascript code.

We've chosen Vue.js because we like <a href="https://vuejs.org/v2/guide/single-file-components.html">Vue's Single File Components</a>, especially how we can encapsulate behaviour and appearance in a custom tag.

You might think "Well, other JS frameworks support componentization too". And that's true. While <a href="https://reactjs.org">React.js</a> seems somewhat equivalent to Vue.js, JSX isn't aesthetically pleasing. This <a href="https://www.reddit.com/r/javascript/comments/8o781t/vuejs_or_react_which_you_would_chose_and_why/e01qn55/">reddit comment</a> explains the difference between Vue.js and React.js very well. <a href="https://www.polymer-project.org">Polymer.js 2.0</a>, which we initially used, just wasn't good. But more on that later in a separate blog post...

The plugin integrates Vue compilation into Playframework using sbt-web's `syncIncremental`; this means every time the code is changed, the code will be re-compiled. The plugin also tracks dependencies within a component (e.g. `import OurButton from './common/our-button.vue'` <a href="https://github.com/GIVESocialMovement/sbt-vuefy/blob/master/test-play-project/app/assets/vue/components/greeting-form.vue">here</a>), and the recompilation will happen only on necessary files.

The data is sent to Javascript code by writing JSON onto the generated HTML. Here's an example of <a href="https://github.com/GIVESocialMovement/sbt-vuefy/blob/master/test-play-project/app/views/index.scala.html">index.scala.html</a> using <a href="https://github.com/GIVESocialMovement/sbt-vuefy/blob/master/test-play-project/app/assets/vue/components/greeting-form.vue">app/assets/vue/components/greeting-form.vue</a>:

{% highlight html %}
@(greeting: String)

<script src="https://cdnjs.cloudflare.com/ajax/libs/vue/2.5.16/vue.js"></script>
<script src='@routes.Assets.versioned("vue/components/greeting-form.js")'></script>

<script>
  function parse(s) {
    return JSON.parse(decodeURIComponent(escape(atob(s))));
  }
</script>

<div id="app"></div>
<script>
  var app = new Vue({
    el: '#app',
    render: function(html) {
      return html(GreetingForm.default, {
        props: {
          greeting: parse("@libraries.Base64.encodeString(greeting)")
        }
      });
    }
  })
</script>
{% endhighlight %}

There are three things to pay attention to in the code above:

* We have to encode JSON in Base64 to prevent <a href="https://www.owasp.org/index.php/Cross-site_Scripting_(XSS)">XSS</a>.
* `greeting-form.vue` is compiled to `greeting-form.js`.
* `GreetingForm.default` comes from the fact that sbt-vuefy uses the camel case of the filename as the exported module. The camel case conversion happens <a href="https://github.com/GIVESocialMovement/sbt-vuefy/blob/master/src/main/resources/sbt-vuefy-plugin.js#L2">here</a>.

A complete example is <a href="https://github.com/GIVESocialMovement/sbt-vuefy/tree/master/test-play-project">here</a>

We have been using <a href="https://github.com/GIVESocialMovement/sbt-vuefy">sbt-vuefy</a> on <a href="https://give.asia">GIVE.asia</a> for some time now, and it has been working well.
Check out <a href="https://github.com/GIVESocialMovement/sbt-vuefy">sbt-vuefy</a> if it might work for you,
and please let us (<a href="https://twitter.com/tanin">@tanin</a>) know if you encounter any problem or have any suggestion!
