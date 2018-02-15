# Importing a JavaScript Module into ClojureScript

An opinionated guide.

## Executive summary

Reccommended approaches in decreasing order of developer convenience:

1. Never do any cljs advanced optimizations. Include your libraries using `<script>` tags. Access methods and properties of the javascript library using the `js/` accessor. There are obvious performance downsides to this approach, but it may be all you need.
2. Replace your lein/cljs build with shadow-cljs. Install your libraries using npm or yarn. Import them directly into your source code. I reccommend this approach if at all possible, but existing projects with complex builds may find this difficult.
3. Use the normal cljs compiler. Find a UMD build from the npm distribution of your library, or create one using webpack if one does not exist. Import it using `:foreign-libs`. Exclusively use `cljs-oops` to access members and functions. Never use the `js/` accessors.

Unreccomended approaches:

1. Use the normal cljs compiler. Hope that a `cljsjs` version of your library exists and use it like a normal cljs library. Hope the author wrapped it properly. Hope they do an update when a new version comes out. If not, create your own externs file either manually or with the automatic externs tool. Be a nice person and submit a pull request to the cljsjs project.
2. Use the `:npm-deps` feature of the cljs compiler. I used it once. It broke with a perplexing error message. There was no way to debug it and nobody on slack ever seems to have answers about it. Although the documentation is clear that this feature is not expected to work all the time, it doesn't say when that might happen or what you are supposed to do about it.

## The problem we are trying to solve

In advanced optimization mode, the Google Closure Compiler alters symbols in javascript code in order to get maximum minification. This is apparently important because the generated cljs code creates incredibly long function names.

Here's the problem. If you access a javascript library like this: `js/LibraryObject.method`, Closure Compiler will come around an minify those symbols and then you'll get a runtime error message like "Cannot read property 'q29' of null". Why? Because Closure Compiler doesn't optimize the external library but it _does_ optimize the cljs code. \(Note, theoretically some javascript libraries are safe to be run through closure advanced optimizations, but in practice, it seems like virtually nothing you'll want to use is.\)

### The "externs" file and cljsjs

The traditional solution is to create an "externs" file that contain symbols the Closure Compiler should _not_ alter. Note, this affects compilation not of the external library, but of the cljs code. \(This was initially confusing to me because the externs feel "attached" to the foreign library.\) So when you write `js/LibraryObject.method`, the `LibaryObject.method` symbol in the compiled cljs code will remain untouched by the advanced optimizer, provided that it is properly declared in the extern files.

The [cljsjs project](https://github.com/cljsjs/packages) has a bunch of libraries that other people have created externs for. These packages also contain `deps.cljs` files, which are little pieces of metadata that instruct the compiler how to load the javascript code and enable you to use a normal namespace when accessing the library \(e.g. `react/createElement`\).

Problem: what if your library isn't there? What if it is there but it's the wrong version? What if it is there but it was packaged incorrectly? Making these libraries involved learning how to use the extern inference tool, learning how to use the `boot` tool, and learning how to write the `deps.cljs` file, and then \(if you are nice\) submitting a pull request.  This is a hell of a lot of friction as compared to `npm install react-flip-move` and just using it.

## Shadow-cljs

[Shadow-cljs](https://shadow-cljs.github.io/docs/UsersGuide.html) is a fork of the main cljs-compiler that allows you to simply install modules using `npm` or `yarn` and just used them. It is primarily a build tool, so aside from some extensions to the `ns` form, it doesn't impact your code much. It accepts a strict superset of cljs code.

Shadow-cljs will replace:

1. The cljs compiler
2. The dependency management portion of lein/boot
3. Figwheel

You can still use lein and boot if your build is more complicated \(like maybe if you have a css preprocessor step\), but for simple projects you don't need them.

#### Quick taste

The shadow guide is thorough, but here is a taste of how it works:

##### npm modules

Use `npm` or `yarn` and install the package normally. Shadow-cljs will look at the `node_modules` directory and examine the information in `package.json` for each module. The module can usually be used directly in your cljs code like so:

```clojure
  (:require [reagent.core :as reagent]
            ["react-dnd" :as react-dnd :refer DropTarget]))
```

* Note: the extensions to `ns.`npm modules can be imported using the same string name that you would use if you were doing an `import FlipMove from "react-flip-move"` in javascript. 

* Note: shadow-cljs supports every conceivable type of es6 import syntax, including default imports. See more [here](https://shadow-cljs.github.io/docs/UsersGuide.html#_using_npm_packages).

##### Figwheel replacement

With a few line so config, you get hot swap code and a heads up display, just like with figwheel.  Your `shadow-cljs.edn` will look something like this:

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
                   :after-load seekeasy.core/reload}
        :js-options {:resolve {"react-flip-move"
                                {:target :npm
                                 :require "react-flip-move/dist/react-flip-move.es"}}}}}
```

A few pointers:

1. The `:devtools` section sets up the equivalent of figwheel. Note the `:after-load` entry point: this is what shadow-cljs will call when it hot reloads new code. Typically, you will just remount the root element in your reagent project, or the equivalent if you are using a different library. Shadow-cljs gives you other hooks so that you can do things such as restarting webworkers.  Either clone or poke around [this repo](https://github.com/lauritzsh/reagent-shadow-cljs-starter) for an example.
2. Some npm modules ship with more than one way to consume it. In the above file I have shown how to consume `react-flip-move` using the es6 module distribution.



