# Importing a JavaScript Module into ClojureScript

My thoughts to date on how best to do this.

## Executive summary

Recommended approaches in decreasing order of developer convenience:

1. Replace your lein/cljs build with shadow-cljs. Install your libraries using npm or yarn. Import them directly into your source code. I recommend this approach if at all possible, but existing projects with complex builds may find this difficult.  More below.
2. Use the normal cljs compiler. Find a UMD build from the npm distribution of your library, or create one using webpack if one does not exist. Import it using `:foreign-libs`. Exclusively use `cljs-oops` to access members and functions. Never use the `js/` accessors.  More on this below.

Other approaches \(not recommended\):

1. Use the normal cljs compiler. Hope that a `cljsjs` version of your library exists and use it like a normal cljs library. Hope the author wrapped it properly. Hope they do an update when a new version comes out. If not, create your own externs file either manually or with the automatic externs tool. Be a nice person and submit a pull request to the cljsjs project.
2. Use the `:npm-deps` feature of the cljs compiler. I used it once. It broke with a perplexing error message. There was no way to debug it and nobody on slack ever seems to have answers about it. Although the documentation is clear that this feature is not expected to work all the time, it doesn't say when that might happen or what you are supposed to do about it.

_Side note:  Are you just doing an internal tool or a learning project?  Then stop reading this guide.  Just never do any cljs advanced optimizations. Include your libraries using _`<script>`_ tags. Access methods and properties of the javascript library using the _`js/`_ accessor. There are obvious performance downsides to this approach, but it is all you need._

## The problems we are trying to solve

1. Accessing JavaScript from ClojureScript
2. Making code save for advanced optimization
3. Packaging the code for delivery to browser

### Problem 1: Accessing Javascript from ClojureScript

The simplest mechanism to access a JavaScript library is to access it via the library's global object.  Before modern module syntax, libraries worked by setting a bunch of properties on a global object.  When you included the library using a normal `<script>` tag, the library would set that global object once it was loaded.  Most libraries come with at least one UMD distribution already \(usually in the /dist directory of their node\_modules installation--you can also pull one off of a site like unpkg\).  The top of it looks something like this:

```
$ head ReactDnD.js
(function webpackUniversalModuleDefinition(root, factory) {
	if(typeof exports === 'object' && typeof module === 'object')
		module.exports = factory(require("react"));
	else if(typeof define === 'function' && define.amd)
		define(["react"], factory);
	else if(typeof exports === 'object')
		exports["ReactDnD"] = factory(require("react"));
	else
		root["ReactDnD"] = factory(root["React"]);
})(this, function(__WEBPACK_EXTERNAL_MODULE_2__) {
```

This hideous code is here to ascertain what environment it is in \(node, AMD, or browswer\) and to install a global object appropriately.  When called inside a `<script>`tag, it will take the last else clause and `root` will be passed in as `window`.  So you can access this library using an object called `ReactDnD`.

You can do this in ClojureScript too, and just access the object using `js/ReactDnD`.  

Problems:

1. To translate from es6-style import syntax, you really need to understand what is going on under the covers
2. You will ship a lot of code to the browser and each library will make a round trip to the server

_Note, with shadow-cljs you import libraries using a real namespace with a syntax that is much more one-to-one with es6 syntax \(described below\)._

### Problem 2: Making code safe for advanced optimization

In advanced optimization mode, the Google Closure Compiler alters symbols in javascript code in order to get maximum minification. This is apparently important because the generated cljs code creates incredibly long function names.

Here's the problem. If you access a javascript library like this: `(.method LibraryObject),` Closure Compiler will come around an minify the `method` symbol and then you'll get a runtime error message like "Cannot read property 'q29' of null". Why? Because Closure Compiler doesn't optimize the external library but it _does_ optimize the cljs code.

Notes:

1. Theoretically some javascript libraries are safe to be run through closure advanced optimizations, but in practice, it seems like virtually nothing you'll want to use is.
2. Automated externs inference in the compiler can be turned on and will solve some of this problem.  But it can't do it for everything.  You can set the `:externs-inference true`to turn it on and set `*warn-on-infer*` on a per-file basis to have it tell you where you need to provide type hints.  Externs inference is turned on by default in `shadow-cljs`.

The "externs" file and cljsjs

The traditional solution is to create an "externs" file that contain symbols the Closure Compiler should _not_ alter. Note, this affects compilation not of the external library, but of the cljs code. \(This was initially confusing to me because the externs feel "attached" to the foreign library.\) So when you write `js/LibraryObject.method`, the `LibaryObject.method` symbol in the compiled cljs code will remain untouched by the advanced optimizer, provided that it is properly declared in the extern files. Note, the cljs compiler has a feature called automated externs inference now, which, when turned on, will automatically avoid changing simple examples like this, but there are other instances where it cannot make the inference, so we have the same problem.

The [cljsjs project](https://github.com/cljsjs/packages) has a bunch of libraries that other people have created externs for. These packages also contain `deps.cljs` files, which are little pieces of metadata that instruct the compiler how to load the javascript code and enable you to use a normal namespace when accessing the library \(e.g. `react/createElement`\).

Problem: what if your library isn't there? What if it is there but it's the wrong version? What if it is there but it was packaged incorrectly? Making these libraries involved learning how to use the extern inference tool, learning how to use the `boot` tool, and learning how to write the `deps.cljs` file, and then \(if you are nice\) submitting a pull request.  This is a hell of a lot of friction as compared to `npm install react-flip-move` and just using it.

## Shadow-cljs

[Shadow-cljs](https://shadow-cljs.github.io/docs/UsersGuide.html) augments the main cljs-compiler with some dependency management, much improved npm module integration, simpler configuration, better defaults, and other conveniences.  With it, you can install modules using `npm` or `yarn` and use them as is. It is primarily a build tool, so aside from some extensions to the `ns` form, it doesn't impact your code. It accepts a strict superset of cljs code.

Shadow-cljs will replace:

1. The cljs compiler \(although it still uses the cljs-compiler under the hood\)
2. The dependency management portion of lein/boot
3. Figwheel \(although you can still use figwheel if you prefer\)

You can still use lein and boot if your build is more complicated \(like maybe if you have a css preprocessor step\), but for simple projects you don't need them.

Shadow-cljs makes npm module integration work better by actually examining the source code of the modules and generating externs in addition to turning the cljs extern inference feature on.

#### Quick taste

The shadow guide is thorough, but here is a taste of how it works:

##### npm modules

Use `npm` or `yarn` and install the package normally. Shadow-cljs will look at the `node_modules` directory and examine the information in `package.json` for each module. The module can usually be used directly in your cljs code like so:

```clojure
  (:require ["react-dnd" :as react-dnd :refer (DropTarget)]))
```

* Note: npm modules can be imported using the same string name that you would use if you were doing an `import FlipMove from "react-flip-move"` in javascript.  \(You can also just use the symbol name `react-flip-move` which I prefer because my linter doesn't know about the string syntax.\)

* Note: shadow-cljs supports every conceivable type of es6 import syntax, including default imports. This makes translation from javascript examples and documentation easy.  See more [here](https://shadow-cljs.github.io/docs/UsersGuide.html#_using_npm_packages).  \(It allows a `:default` keyword that is technically an extension to the allowed syntax.\)

##### Figwheel replacement

With a few lines of config, you get hot code reloading and a heads up display, just like with figwheel.  Your `shadow-cljs.edn` will look something like this:

```clojure
{:source-paths
 ["src"]

 :dependencies
 [[org.clojure/clojure "1.9.0"]
  [org.clojure/clojurescript "1.9.946"]
  [reagent "0.7.0"]
  ...more stuff here]

 :builds
 {:app {:target :browser
        :output-dir "public/js"
        :asset-path "/js"
        :modules {:main {:entries [myapp.core]}}
        :devtools {:http-root "public"
                   :http-port 3000
                   :http-handler shadow.http.push-state/handle
                   :after-load seekeasy.core/reload}}}
```

A few pointers:

1. The `:devtools` section sets up the equivalent of figwheel. Note the `:after-load` entry point: this is what shadow-cljs will call when it hot reloads new code. Typically, you will just remount the root element in your reagent project, or the equivalent if you are using a different library. 
2. Shadow-cljs gives you other hooks so that you can do things such as restarting webworkers.  Either clone or poke around [this repo](https://github.com/lauritzsh/reagent-shadow-cljs-starter) for an example.

##### Minification, bundling, and selective imports

Shadow-cljs kind of operates like webpack for you.  It will bundle and perform safe minifications on all your javascript libraries.  On slack, I saw one project that ported to shadow-cljs and saw big improvements in bundle size without doing anything.

One advantage of supporting the full es6 import syntax is that some libraries are built with components that can be imported individually \(e.g. material-ui\).  You can potentially see big improvements in code size just because of this feature.

##### Rough spots

So far, I found that we had to give a hint to the compiler when doing a es6 style default import statements with the `react-flip-move` package, because it was picking up the wrong distributed entry point.  All we had to do, though, was tell shadow-cljs which of the handful of distributions it ships with to use:

```
:js-options {:resolve {"react-flip-move"
                                {:target :npm
                                 :require "react-flip-move/dist/react-flip-move.es"}}}}}
```

In at least one case, the `ns` extensions will break tooling that depends on parsing those forms \(in my case, using `joker` as a linter\).  I have usually been able to write the `ns` forms in such a was as to avoid the extensions \(such as using symbols instead of strings\).

The compiler is under active development so if you run into trouble you can get help and get fix if needed surprisingly fast.

##### Summary

If you are doing a lot of javascript interop, shadow-cljs is easier, more idiomatic for javascript libraries, and it is well supported.  Come by \#shadow-cljs in slack if you have issues.

## Normal cljs compiler and cljs-oops

The other technique that avoids the externs and cljsjs friction is to stick with the mainstream compiler and use the `cljs-oops` package to access the libraries.  The advantage here is that you don't stray from the herd and your existing tooling will work.  The disadvantage is that it's a little bit clunky in your source code and you have to be diligent never to access javascript objects using the `js/` accessor.

The first step is to find a UMD build of your package.  Typically, you can just `npm install` the package, go to `node_modules,`look inside the `dist` directory, and grab the build from there.  The top of the file should look like the UMD example shown above.

Using that example, all you need to do now is inform the compiler that you want this included:

```
:foreign-libs [{:file "resources/lib/ReactDnD.js"
                :file-min "lib/ReactDnD.min.js"
                :provides ["react-dnd"]}]
```

Now, if we just do something like `(.dropTarget monitor)` the optimizer might alter the `dropTarget`symbol and you'll get a runtime error in production.  Instead, we use the [cljs-oops](https://github.com/binaryage/cljs-oops) library: `(ocall dnd-connect "dropTarget")` or like this `(ocall dnd-connect :dropTarget)` Because we are using strings or keywords, the optimizer will not alter this code.  You can use `oget` and `oset` as well for properties.  The library has a bunch of convenience macros and some error checking built in to to make this less error prone.

That's it.  As long as you consistently use `cljs-oops` when you refer to foreign lib symbols, everything will work.

##### Summary

This technique works, but it requires you to do some investigation to figure out how your library is packaged and how it exports its global object. If the examples are using sophisticated es6 import syntax, that may require some translation on your part.  You don't get npm module resolution.  You have to keep to a convention when you call javascript code.  But it is still better than messing around with externs, and you may be more comfortable using the standard tool set.

```clojure

```



