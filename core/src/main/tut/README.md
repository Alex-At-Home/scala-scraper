# Scala Scraper [![Build Status](https://travis-ci.org/ruippeixotog/scala-scraper.svg?branch=master)](https://travis-ci.org/ruippeixotog/scala-scraper) [![Coverage Status](https://coveralls.io/repos/github/ruippeixotog/scala-scraper/badge.svg?branch=master)](https://coveralls.io/github/ruippeixotog/scala-scraper?branch=master) [![Maven Central](https://img.shields.io/maven-central/v/net.ruippeixotog/scala-scraper_2.11.svg)](https://maven-badges.herokuapp.com/maven-central/net.ruippeixotog/scala-scraper_2.11) [![Join the chat at https://gitter.im/ruippeixotog/scala-scraper](https://badges.gitter.im/ruippeixotog/scala-scraper.svg)](https://gitter.im/ruippeixotog/scala-scraper)

A library providing a DSL for loading and extracting content from HTML pages.

Take a look at [Examples.scala](https://github.com/ruippeixotog/scala-scraper/blob/master/src/test/scala/net/ruippeixotog/scalascraper/Examples.scala) and at the [unit specs](https://github.com/ruippeixotog/scala-scraper/tree/master/src/test/scala/net/ruippeixotog/scalascraper) for usage examples or keep reading for more thorough documentation. Feel free to use [GitHub Issues](https://github.com/ruippeixotog/scala-scraper/issues) for submitting any bug or feature request and [Gitter](https://gitter.im/ruippeixotog/scala-scraper) to ask questions.

## Quick Start

To use Scala Scraper in an existing SBT project with Scala 2.11 or 2.12, add the following dependency to your `build.sbt`:

```scala
libraryDependencies += "net.ruippeixotog" %% "scala-scraper" % "1.2.1"
```

If you are using an older version of this library, see this document for the version you're using: [0.1](https://github.com/ruippeixotog/scala-scraper/blob/v0.1/README.md), [0.1.1](https://github.com/ruippeixotog/scala-scraper/blob/v0.1.1/README.md), [0.1.2](https://github.com/ruippeixotog/scala-scraper/blob/v0.1.2/README.md).

An implementation of the `Browser` trait, such as `JsoupBrowser`, can be used to fetch HTML from the web or to parse a local HTML file or a string:

```tut:book:silent
import net.ruippeixotog.scalascraper.browser.JsoupBrowser

val browser = JsoupBrowser()
val doc = browser.parseFile("core/src/test/resources/test2.html")
val doc2 = browser.get("http://example.com")
```

The returned object is a `Document`, which already provides several methods for manipulating and querying HTML elements. For simple use cases, it can be enough. For others, this library improves the content extracting process by providing a powerful DSL.

You can open the [test2.html](core/src/test/resources/test2.html) file loaded above to follow the examples throughout the README.

First of all, the DSL methods and conversions must be imported:

```tut:book:silent
import net.ruippeixotog.scalascraper.dsl.DSL._
import net.ruippeixotog.scalascraper.dsl.DSL.Extract._
import net.ruippeixotog.scalascraper.dsl.DSL.Parse._
```

Content can then be extracted using the `>>` extraction operator and CSS queries:

```tut:book
import net.ruippeixotog.scalascraper.model._

// Extract the text inside the element with id "header"
doc >> text("#header")

// Extract the <span> elements inside #menu
val items = doc >> elementList("#menu span")

// From each item, extract all the text inside their <a> elements
items.map(_ >> allText("a"))

// From the meta element with "viewport" as its attribute name, extract the
// text in the content attribute
doc >> attr("content")("meta[name=viewport]")
```

If the element may or may not be in the page, the `>?>` tries to extract the content and returns it wrapped in an `Option`:

```tut:book
// Extract the element with id "footer" if it exists, return `None` if it
// doesn't:
doc >?> element("#footer")
```

With only these two operators, some useful things can already be achieved:

```tut:book:silent
// Go to a news website and extract the hyperlink inside the h1 element if it
// exists. Follow that link and print both the article title and its short
// description (inside ".lead")
for {
  headline <- browser.get("http://observador.pt") >?> element("h1 a")
  headlineDesc = browser.get(headline.attr("href")) >> text(".lead")
} println("== " + headline.text + " ==\n" + headlineDesc)
```

In the next two sections the core classes used by this library are presented. They are followed by a description of the full capabilities of the DSL, including the ability to parse content after extracting, validating the contents of a page and defining custom extractors or validators.

## Core Model

The library represents HTML documents and their elements by [Document](https://github.com/ruippeixotog/scala-scraper/blob/master/src/main/scala/net/ruippeixotog/scalascraper/model/Document.scala) and [Element](https://github.com/ruippeixotog/scala-scraper/blob/master/src/main/scala/net/ruippeixotog/scalascraper/model/Element.scala) objects, simple interfaces containing several methods for retrieving information and navigating throughout the DOM.

[Browser](https://github.com/ruippeixotog/scala-scraper/blob/master/src/main/scala/net/ruippeixotog/scalascraper/browser/Browser.scala) implementations are the entrypoints for obtaining `Document` instances. They implement most notably `get`, `post`, `parseFile` and `parseString` methods for retrieving documents from different sources. Depending on the browser used, `Document` and `Element` instances may have different semantics, mainly on their immutability guarantees.

## Browsers

The library currently provides two built-in implementations of `Browser`:

* [JsoupBrowser](https://github.com/ruippeixotog/scala-scraper/blob/master/src/main/scala/net/ruippeixotog/scalascraper/browser/JsoupBrowser.scala) is backed by [jsoup](http://jsoup.org/), a Java HTML parser library. `JsoupBrowser` provides powerful and efficient document querying, but it doesn't run JavaScript in the pages. As such, it is limited to working strictly with the HTML send in the page source;
* [HtmlUnitBrowser](https://github.com/ruippeixotog/scala-scraper/blob/master/src/main/scala/net/ruippeixotog/scalascraper/browser/HtmlUnitBrowser.scala) is based on [HtmlUnit](http://htmlunit.sourceforge.net), a GUI-less browser for Java programs. `HtmlUnitBrowser` simulates thoroughly a web browser, executing JavaScript code in the pages besides parsing and modelling its HTML content. It supports several compatibility modes, allowing it to emulate browsers such as Internet Explorer.

Due to its speed and maturity, `JsoupBrowser` is the recommended browser to use when JavaScript execution is not needed. More information about each browser and its semantics can be obtained in each implementations' Scaladoc.

## Content Extraction

The `>>` and `>?>` operators shown above accept an `HtmlExtractor` as their right argument, which has a very simple interface:

```scala
trait HtmlExtractor[-E <: Element, +A] {
  def extract(doc: ElementQuery[E]): A
}
```

One can always create a custom extractor by implementing `HtmlExtractor`. However, the DSL provides several useful constructors for `HtmlExtractor` instances. In general, you can use the `extractor` factory method:

```
doc >> extractor(<cssQuery>, <contentExtractor>, <contentParser>)
```

Where the arguments are:

* **cssQuery**: the CSS query used to select the elements to be processed;
* **contentExtractor**: defines the content to be extracted from the selected elements, e.g. the element objects themselves, their text, a specific attribute, form data;
* **contentParser**: an optional parser for the data extracted in the step above, such as parsing numbers and dates or using regexes.

The DSL provides several `contentExtractor` and `contentParser` instances, which were imported before with `DSL.Extract._` and `DSL.Parse._`. The full list can be seen in the `ContentExtractors` and `ContentParsers` objects inside [HtmlExtractor.scala](https://github.com/ruippeixotog/scala-scraper/blob/master/src/main/scala/net/ruippeixotog/scalascraper/scraper/HtmlExtractor.scala).

Some usage examples:

```tut:book
// Extract the date from the "#date" element
doc >> extractor("#date", text, asLocalDate("yyyy-MM-dd"))

// Extract the text of all "#mytable td" elements and parse each of them as a number
doc >> extractor("#mytable td", texts, seq(asDouble))

// Extract an element "h1" and do no parsing (the default parsing behavior)
doc >> extractor("h1", element, asIs[Element])
```

The DSL also provides implicit conversions to write more succinctly the most common extractor types:

* Writing `<contentExtractor>(<cssQuery>)` is taken as `extractor(<cssQuery>, <contentExtractor>, asIs)`;
* Writing `<cssQuery>` is taken as `extractor(<cssQuery>, elements, asIs)`.

That way, one can write the expressions in the Quick Start section, as well as:

```tut:book
// Extract the elements with class ".active"
doc >> elementList(".active")

// Extract the text inside each p element
doc >> texts("p")

// Exactly the "h3" elements (as a lazy iterable)
doc >> "h3"
```

## Content Validation

While scraping web pages, it is a common use case to validate if a page effectively has the expected structure. This library provides special support for creating and applying validations.

A `HtmlValidator` has the following signature:

```scala
trait HtmlValidator[-E <: Element, +R] {
  def matches(doc: ElementQuery[E]): Boolean
  def result: Option[R]
}
```

As with extractors, the DSL provides the `validator` constructor and the `>/~` operator for applying a validation to a document:

```
doc >/~ validator(<extractor>)(<matcher>)
```

Where the arguments are:

* **extractor**: an extractor as defined in the previous section;
* **matcher**: a function mapping the extracted content to a boolean indicating if the document is valid.

The result of a validation is an `Either[R, A]` instance, where `A` is the type of the document and `R` is the result type of the validation (which will be explained later).

Some validation examples:

```tut:book
// Check if the title of the page is "Test page"
doc >/~ validator(text("title"))(_ == "Test page")

// Check if there are at least 3 ".active" elements
doc >/~ validator(".active")(_.size >= 3)

// Check if the text in ".desc" contains the word "blue"
doc >/~ validator(allText("#mytable"))(_.contains("blue"))
```

When a document fails a validation, it may be useful to identify the problem by pattern-matching it against common scraping pitfalls, such as a login page that appears unexpectedly because of an expired cookie, dynamic content that disappeared or server-side errors. If we define validators for both the success case and error cases:

```tut:book:silent
val succ = validator(text("title"))(_ == "My Page")

val errors = Seq(
  validator(allText(".msg"), "Not logged in")(_.contains("sign in")),
  validator(".item", "Too few items")(_.size < 3),
  validator(text("h1"), "Internal Server Error")(_.contains("500")))
```

They can be used in combination to create more informative validations:

```tut:book
doc >/~ (succ, errors)
```

Validators matching errors were constructed above using an additional `result` parameter after the extractor. That value is returned wrapped in a `Left` if that particular error occurs during a validation.

## Other DSL Features

As shown before in the Quick Start section, one can try if an extractor works in a page and obtain the extracted content wrapped in an `Option`:

```tut:book
// Try to extract an element with id "optional", return `None` if none exist
doc >?> element("#optional")
```

Note that when using `>?>` with content extractors that return sequences, such as `texts` and `elements`, `None` will never be returned (`Some(Seq())` will be returned instead).

If you want to use multiple extractors in a single document or element, you can pass tuples or triples to `>>`:

```tut:book
// Extract the text of the title element and all inputs of #myform
doc >> (text("title"), elementList("#myform input"))
```

The extraction operators work on `List`, `Option`, `Either` and other instances for which a [Scalaz](https://github.com/scalaz/scalaz) `Functor` instance is provided. The extraction occurs by mapping over the functors:

```tut:book
// Extract the titles of all documents in the list
List(doc, doc) >> text("title")

// Extract the title if the document is a `Some`
Option(doc) >> text("title")
```

You can apply other extractors and validators to the result of an extraction, which is particularly powerful combined with the feature shown above:

```tut:book
// From the "#menu" element, extract the text in the ".active" element inside
doc >> element("#menu") >> text(".active")

// Same as above, but in a scenario where "#menu" can be absent
doc >?> element("#menu") >> text(".active")

// Same as above, but check if the "#menu" has any "span" element before
// extracting the text
doc >?> element("#menu") >/~ validator("span")(_.nonEmpty) >> text(".active")

// Extract the links inside all the "#menu > span" elements
doc >> elementList("#menu > span") >?> attr("href")("a")
```

This library also provides a `Functor` for `HtmlExtractor`, which makes it possible to map over extractors and create chained extractors that can be passed around and stored like objects. For example, new extractors can be defined like this:

```tut:book:silent
import net.ruippeixotog.scalascraper.scraper.HtmlExtractor

// An extractor for the links inside all the "#menu > span" elements
val linksExtractor: HtmlExtractor[Element, List[Option[String]]] =
  elementList("#menu > span") >?> attr("href")("a")

// An extractor for the number of links
val linkCountExtractor: HtmlExtractor[Element, Int] =
  linksExtractor.map(_.flatten.length)
```

And they can be used just as extractors created using other means provided by the DSL:

```tut:book
doc >> linksExtractor

doc >> linkCountExtractor
```

Just remember that you can only apply extraction operators `>>` and `>?>` to documents, elements or functors "containing" them, which means that the following is a compile-time error:

```tut:book:fail
// The `texts` extractor extracts a list of strings and extractors cannot be
// applied to strings
doc >> texts("#menu > span") >> "a"
```

Finally, if you prefer not using operators for the sake of code legibility, you can use full method names:

```tut:book
// `extract` is the same as `>>`
doc extract text("title")

// `tryExtract` is the same as `>?>`
doc tryExtract element("#optional")
```

## Integration with Typesafe Config

The [Scala Scraper Config module](modules/config/README.md) can be used to load extractors and validators from config files.

## Working Behind an HTTP/HTTPS Proxy

If you are behind an HTTP proxy, you can configure `Browser` implementations to make connections through it by setting the Java system properties `http.proxyHost`, `https.proxyHost`, `http.proxyPort` and `https.proxyPort`. Scala Scraper provides a `ProxyUtils` object that facilitates that configuration:

```tut:book:silent
import net.ruippeixotog.scalascraper.util.ProxyUtils

ProxyUtils.setProxy("localhost", 3128)
val browser = JsoupBrowser()
// HTTP requests and scraping operations...
ProxyUtils.removeProxy()
```

`JsoupBrowser` uses internally `java.net.HttpURLConnection`. Configuring those JVM-wide system properties will affect not only `Browser` instances, but _all_ requests done using `HttpURLConnection` directly or indirectly. `HtmlUnitBrowser` was implementated so that it reads the same system properties for configuration, but once the browser is created they will be used on every request done by the instance, regardless of the properties' values at the time of the request.

_NOTE: this feature is in a beta stage. Please expect API changes in future releases._

## Copyright

Copyright (c) 2014-2017 Rui Gonçalves. See LICENSE for details.
