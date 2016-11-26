---
layout: post
title: What are externs, and should you care?
---

For better or worse, the word "externs" has made its way into my radar. The first time I heard it used was when a coworker was explaining why Closure made it difficult to introduce React and other third party JS libraries. The second time I saw it used was when I was investigating how to use third party JS libraries in Clojurescript.

Anyway, I gleaned that externs were related to the [Google Closure compiler](https://github.com/google/closure-compiler), and tried to figure out why they kept coming up in conversation.

*Disclaimer: I have never worked with the Closure compiler in production and I have limited professional Javascript experience. I just happen to be curious about the Closure compiler and Javascript technologies in general.*

### What is Google Closure compiler?
For several months, all I knew about Google's Closure compiler was that it was a Javascript minifier capable of "tree-shaking" - that is, removing unused code from a Javascript file.

However, I discovered that Google's Closure compiler shouldn't be thought of as minifier. It's more accurately described as a compiler whose only reason of existing is to take large JS files and make them smaller.

### What kind of code does Closure compile?
Short answer: ES5 Javascript with some ES6 support.

Long answer: ES5 Javascript with some ES6 support, but depending on the compilation mode selected, your program won't work if you engage in the practices listed [here](https://developers.google.com/closure/compiler/docs/limitations). In whitespace mode or simple mode, Closure basically compiles ES5 JS. But in advanced mode operating within a browser environment, the Closure compiler compiles ES5 JS with some important constraints.

### Why do people use Closure?

The Closure compiler offers three compilation modes: `WHITESPACE_ONLY`, `SIMPLE_OPTIMIZATIONS`, and `ADVANCED_OPTIMIZATIONS`. The `ADVANCED_OPTIMIZATIONS` mode can drastically reduce the size of a JS application because it renames global variables, function names, and properties. It also removes code that is provably unreachable, which means the output JS only contains functions and code that you're actually using.

Websites are interested in reducing the size of their JS because that way, they can be delivered to the client and parsed faster. The faster JS is loaded, the faster a website become interactive.

Nowadays, many people use [Uglify](https://github.com/mishoo/UglifyJS) to minify their ES5 Javascript. But Closure, a mature Java beast maintained by Google, [produces JS files that are significantly smaller than that of its competitors](https://nolanlawson.com/2016/08/15/the-cost-of-small-modules/) (in advanced mode).

### What's the catch? Why doesn't everyone use Closure?

There's no point in using Closure unless you take advantage of the optimizations offered in advanced mode. However, there's no point in incurring the mental overhead of writing Closure-advanced-mode-friendly code and introducing a compilation stage to your build pipeline and dev environment unless your Javascript app is big and could use the significant file size savings promised by Closure.

**The key thing to remember:** The Closure compiler is only able to apply massive optimizations to JS written with the specific constraints it demands in advanced mode. So, the higher percent of advanced-mode-compatible code in a Closure project, the more savings there are to be had.

Google knows that Closure is used by websites to output small JS for browsers. Thus, it created the Google Closure library, a collection of bread and butter JS libraries *designed to be integrated with Closure projects using advanced compilation mode*. In other words, the Google Closure library follows all the rules necessary to be successfully included in the input set of an advanced mode Closure compilation.

Okay, but what if your Closure project wants some functionality not provided by anything in the Closure library? A common case would be DOM construction and manipulation, [a problem to which the JS community has many answers](http://stateofjs.com/2016/frontend/). Unforuntately, none of these JS frameworks, and very few JS libraries in general, are served in the flavor of JS suitable to be included in Closure advanced mode compilations.

Of course, you don't need to put *every* line of JS into your advanced mode compilation set, though it would certainly maximize compression. The Closure compiler offers an escape hatch that enables advanced-mode-compiled JS to interact with uncompiled code - via the declaration of externs.

### Enter externs

The Closure compiler recommends the following strategy for [interacting with uncompiled code from within advanced mode compiled Closure code](https://developers.google.com/closure/compiler/docs/api-tutorial3):

* Load the uncompiled JS file, such as a jQuery distribution, into the browser by way of a `<script>` tag before the Closure compiled code is loaded.
* Write Closure flavored JS using whatever objects and functions you need from the uncompiled JS file, like `jQuery('#id').html()`.
* While compiling your Closure JS, let the Closure compiler know about all the objects and properties it shouldn't rename during compilation. This is done by providing Closure an externs file, and one that would be needed for the example above would look like this:

```javascript
function jQuery(arg1, arg2) {};
jQuery.prototype.html = function(arg1) {};
```

By providing this externs file to the the Closure compiler, you're telling it: "Besides the compiled code you're providing, I'm also including some external code that exposes some functions and objects (listed in my externs file) into the browser. I'll be using these objects and functions from within my Closure code, so would you remember not to rename them during compilation?"

Inside the Google Closure repo, you'll find a folder of [externs for common JS libraries](https://github.com/google/closure-compiler/tree/master/contrib/externs). Community members also publish externs for JS libraries they're using on GitHub.

### "Five weird extern tricks Closure maintainers HATE!"

Not five, but I've stumbled upon two so far:

* A valid extern file for an unminified JS distribution [is the file itself](http://swannodette.github.io/2014/03/14/externs-got-you-down)! This works because the distribution already lists all the functions it exposes.
* Even in advanced mode, the Closure compiler does not rename, minify or otherwise alter string literals. So, you could call a function like `foo["bar"]()` from within Closure code and not have to declare an extern for the `bar` function.

### How do you produce an externs file?
From what I can tell, the current approach to generating an externs file for a library from scratch is to:

* run the file through a tool like [this](https://github.com/jmmk/javascript-externs-generator) (suggested by a ClojureScript wiki page) or [this](http://www.dotnetwise.com/Code/Externs/index.html) (suggested by a Google Closure wiki page)
* see if the automatically generated externs work by writing some automated tests
* amend the file as necessary

I suspect there is no silver bullet that automatically generates externs from any given JS file and guarantees the library's entire public API can be safely used. I'm not saying this because I've looked into it or anything, I just think that if it were easy someone would have done it already!

### Putting things into perspective

As I mentioned earlier, people don't typically consider Closure unless they have heaps and heaps of JS in their projects that would take an unacceptable amount of time to deliver to clients uncompressed. For what it's worth, I only know of three organizations using Closure in production: Google, Yelp, and Medium. Maybe things were different in the past, but my guess is that most JS developers in 2016 do not use Closure at work or for fun.

The first time I heard about Closure was when a few colleagues were discussing why it wasn't a good fit for JS at Yelp. I can't speak to how Closure is received at other organizations. Based purely on my impressions, it seems Closure is a necessary evil if you need that degree of JS compression.

When I say necesary evil, I'm referring to the specific flavor of JS you need to write (not including externs) to be advanced-mode-compilable. However, I'm sure some people find the type annotations and dependency system an *advantage* over writing vanilla ES5. Personally, I find Javascript and all its permutations (ES5, ES6, Closure) painfully confusing so I've adopted alternatives like [Elm](http://elm-lang.org/) and [Clojurescript](http://clojurescript.org/).

### Clojurescript and Closure
I've referenced Clojurescript multiple times in this post, so for the unacquainted, Clojurescript is a [Clojure](http://clojure.org/) dialect that compiles to Javascript.

Because Clojurescript is supposed to be the same as Clojure except compiling to JS instead Java, its needs to support a large set of core functions and implement a bunch of immutable data structures on top of JS. To prevent Clojurescript users from needing to include a potentially enormous blob of JS in their sites, the Clojurescript designers built the language to emit JS for the Closure compiler in advanced mode. As a result, Clojurescript users can sleep easy knowing unused parts of the Clojurescript core will be trimmed away by Closure's dead code elimination.

Although Closure maintainers probably weren't thinking of this use case while writing the compiler, I think it's worth mentioning Clojurescript when discussing Closure since it's a rather intriguing application of the compiler.

Even though I've more or less spurned Javascript as a language I'd write myself, I'm always impressed by the degree of innovation happening in and around the Javascript community.
