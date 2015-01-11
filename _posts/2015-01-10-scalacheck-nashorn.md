---
layout: post
title: Testing javascript with ScalaCheck and Nashorn
---

We all have to write javascript, but javascript has warts and I at least write tons of bugs when I try to implement anything complex - the only way I know to mitigate this is to write tests. However, to get 100% coverage of complex code can often take 2x the original lines of code in unit tests - all of which must be kept up to date and maintained. Here, property-based testing is a lifesaver, allowing you to generate many more test cases than you could by hand with minimumal coding. ScalaCheck is a fantastic property-based testing library. (see [here](https://github.com/rickynils/scalacheck/wiki/User-Guide) for an introduction to property-based testing and ScalaCheck).
Why not use ScalaCheck to test your javascript?  ScalaCheck can generate the test cases and apply them to your production javascript code, thus avoiding the need to write error-prone tests in javascript.

I decided to try this out on our current project, a data synchronization framework.  We have around 1500 lines of rather abstract and difficult-to-reason-about coffeescript code which, though it has a collection of over 40 passing unit tests, has been repeatedly shown to have problematic edge cases in production.  Time to thoroughly clean out all those bugs with ScalaCheck!  After only 100 lines of code we unearthed 10 latent bugs in our production code... and one in the JVM!

To avoid going into depth on how the data synchronization library works, I'll instead show you a simple test that finds the JVM bug - if Oracle had used similar property-based testing against their own code, the bug never would have made it to production.  We will be testing the following function, which should compact a json document by parsing it and regenerating, stored in `target.js`.

{% highlight javascript linenos %}
function compactify(json) { 
  return JSON.stringify(JSON.parse(json))
}
{% endhighlight %}

The first step is to generate the test data using ScalaCheck's Gen library.  Fortunately, there are already generators available for arbitrary Json ***here*** by mixing in the JsonGen trait to our spec.

{% highlight scala linenos %}
import org.scalacheck._
import Prop.forAll

object JsonParseTest extends Properties("Nashorn JSON.parse") with JsonGen {
  property("reverses JSON.stringify") =
    forAll(jsValue(maxDepth = 1)) { value =>
      val json = value.toString         // JSON string
      val compactified = ?              // Call our javascript function
      Json.parse(compactified) == value // Should return the same JsValue as before
    }
}
{% endhighlight %}

Now we need a method of calling into javascript.  You may choose to use an external tool such as PhantomJS or NodeJS but the JVM has had a built-in javascript engine known as Rhino for many years, and in Java 8 this has been replaced with a brand-new modern engine known as Nashorn.  This offers tighter integration with Java but doesn't provide any of the global objects such as `window` or `console` which much javascript relies on.  If your code doesn't use these, or you can easily mock them out, Nashorn is a great choice for running tests in the JVM.

From the Oracle documentation adapted to Scala, you can set up an engine in openJDK 8 as follows:

{% highlight scala linenos %}
val engine = new ScriptEngineManager(null).getEngineByName("nashorn")
{% endhighlight %}


The `null` classloader parameter is a gotcha for the openjdk.. without it [getEngineByName returns null](http://stackoverflow.com/questions/20168226/sbt-0-13-scriptengine-is-null-for-getenginebyname-javascript).

We then need to load in our javascript for testing

{% highlight scala %}
engine.eval(new FileReader("target.js"))
{% endhighlight %}

Finally, we can call a function in the engine so:

{% highlight scala %}
val compactified = engine.asInstanceOf[Invocable]
                         .invokeFunction("compactify", json)
                         .asInstanceOf[String]
{% endhighlight %}

Now we need to run the tests. `sbt` can automagically pick up and run scalacheck tests.  Add the following dependency to your Build.scala

{% highlight scala %}
"org.scalacheck" %% "scalacheck" % "1.11.4" % "test",
{% endhighlight %}

By default ScalaCheck generates 100 values for each test, but more will probably be needed to find this bug.  Add the following to your sbt settings:

{% highlight scala %}
testOptions in Test += Tests.Argument(TestFrameworks.ScalaCheck, "-minSuccessfulTests", "10000")
{% endhighlight %}

And run it!

    sbt test

ScalaCheck will now proceed to generate truck-loads of json and throw it at the method.  Surprisingly, it throws up a problem right away:

    [info] ! Op.JSON can be serialized to the jvm: Falsified after 287 passed tests.
    [info] > ARG_0: {"5":1,"1":"rfIdufb0"}

Can it really be that there's a mistake in the implementation of JSON.parse in Nashorn? If you run the following in build 40 of openjdk 8.0 you'll see that there really is. 

    $ jjs
    jjs> JSON.stringify(JSON.parse('{"5":1,"1":"rfIdufb0"}'));
    {"1":"rfIdufb0"}
    
I've submitted this so hopefully by the time you read this it'll be fixed. Kudos ScalaCheck.

