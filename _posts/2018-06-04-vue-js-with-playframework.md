---
layout: post
title: "Vue.js with Playframework"
date: 2018-04-06 16:24:00 -08:00
author:
  name: tanin
  url: https://twitter.com/tanin
---

We've finally extracted out our Vue.js plugin for Playframework, <a href="https://github.com/GIVESocialMovement/sbt-vuefy">sbt-vuefy</a>, and shared it with the world.

It has been quite an effort to make Vue.js work seamlessly well with Playframework. This is especially true when we want to avoid using <a href="https://en.wikipedia.org/wiki/Single-page_application">Single-page application (SPA)</a>.

<!---excerpt--->

At <a href="https://give.asia">GIVE.asia</a>, we've decided to go with Server-side rendering, the traditional way, as opposed to Client-side rendering or SPA. Because our site isn't highly interactive, and, with SPA, some website's fundamentals (e.g. SEO, Routing) become complex
and requires a lot more Javascript code.

We've chosen Vue.js because we like <a href="https://vuejs.org/v2/guide/single-file-components.html">Vue's Single File Components</a>, especially how we can encapsulate behaviour and appearance in a custom tag.

At this point, you might think "Well, other JS frameworks support componentization too". And that's true. While <a href="https://reactjs.org">React.js</a> seems somewhat equivalent to Vue.js, JSX isn't aesthetically pleasing. <a href="https://www.polymer-project.org">Polymer.js 2.0</a>, which we initially used, just wasn't good. But more on that later in a separate blog post.

Back to <a href="https://github.com/GIVESocialMovement/sbt-vuefy">sbt-vuefy</a>, we integrate the Vue compilation using sbt-web's `syncIncremental`; this means every time the code is changed, the code will be re-compiled. The plugin also tracks dependencies within a component (e.g. `import OurButton from './common/our-button.vue'` <a href="https://github.com/GIVESocialMovement/sbt-vuefy/blob/master/test-play-project/app/assets/vue/components/greeting-form.vue">here</a>), and the recompilation happens on necessary files as one expects.

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

Now you might notice two odd things in the code above:

* We have to encode JSON in Base64 to prevent <a href="https://www.owasp.org/index.php/Cross-site_Scripting_(XSS)">XSS</a>
* `GreetingForm.default` comes from the fact that sbt-vuefy uses the camel case of the filename as the exported module. The camel case conversion happens <a href="https://github.com/GIVESocialMovement/sbt-vuefy/blob/master/src/main/resources/sbt-vuefy-plugin.js#L2">here</a>. It would be great if we can reduce this kind of verbosity.

Check out <a href="https://github.com/GIVESocialMovement/sbt-vuefy">sbt-vuefy</a> and let us know if it helps solve your problem like it solves ours!
