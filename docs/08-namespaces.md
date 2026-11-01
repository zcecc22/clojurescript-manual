# 08 — Namespaces

*After this chapter you can split code across namespaces, require ClojureScript, Closure and Node modules correctly, and share code with JVM Clojure through `.cljc` files.*

---

## One namespace per file

Every file starts with an `ns` form that names it and declares what it depends on:

```clojure
(ns app.util.text
  (:require [clojure.string :as str]))

(defn shout [s] (str (str/upper-case s) "!"))
(defn- helper [] :private)
(def version "1.0")
```

The name must match the path under a source directory: `app.util.text` lives at `src/app/util/text.cljs`, with hyphens becoming underscores (chapter 01).

Everything defined with `def` or `defn` is public by default. `defn-` (and `^:private` on a `def`) marks something as private.

> ⚠️ **Gotcha:** Privacy is **not enforced** in ClojureScript. Calling `text/helper` from another namespace compiles with no warning and works. Treat `defn-` as documentation of intent, the same as a leading underscore in Python.

---

## `:require`

```clojure
(ns app.main
  (:require [app.util.text :as text]                 ; alias
            [app.util.text :refer [shout]]           ; bring names in unqualified
            [clojure.set :refer [union] :rename {union set-union}]
            [app.shared :as shared]))

(text/shout "hi")         ;; => "HI!"
(shout "yo")              ;; => "YO!"
text/version              ;; => "1.0"
(set-union #{1} #{2})     ;; => #{1 2}
```

Prefer `:as` with a short alias. Reading `str/join` tells you where `join` comes from, while a bare `join` brought in by `:refer` doesn't. Keep `:refer` for a few names you use constantly, like `deftest` and `is` in tests.

Conventional aliases are worth following, since readers recognise them: `clojure.string :as str`, `clojure.set :as set`, `clojure.walk :as walk`, `clojure.edn :as edn`, `cljs.pprint :as pprint`.

> **Coming from imperative languages:** There's no wildcard import and no `use`-everything. Like Go, every dependency is explicit and every external name is qualified. Unlike Go, the alias is chosen at the import site, not fixed by the package.

### Order of definition

A namespace is evaluated top to bottom, so a function must be defined before it's used. `declare` creates a forward reference when you need mutual recursion or want to put helpers below their callers:

```clojure
(declare later)

(defn main [& _] (later))
(defn later [] :declared-before-use)
```

### No circular dependencies

Namespaces form a DAG. A cycle is a build error:

```
Circular dependency detected: app.a -> app.b -> app.a
```

The usual fix is to move the shared code into a third namespace that both require.

---

## Requiring Node modules

shadow-cljs lets you require JavaScript modules by **string** name. Node built-ins use the `node:` prefix:

```clojure
(ns app.main
  (:require ["node:path" :as path]
            ["node:fs" :as fs]
            ["node:os" :refer [platform]]
            ["node:events" :refer [EventEmitter]]))

(path/join "a" "b" "c.txt")    ;; => "a/b/c.txt"
(path/extname "x.cljs")        ;; => ".cljs"
(fs/existsSync "shadow-cljs.edn")  ;; => true
(platform)                     ;; => "linux"
```

`:as` binds the module's exports object, and `alias/name` reads a property of it. `:refer` pulls named exports in directly. npm packages work the same way, `["some-package" :as pkg]`, though this manual uses only built-ins.

> ⚠️ **Gotcha:** `:default` imports a module's ES *default export*. Node's built-in modules are CommonJS and have none, so in a `:node-script` build
>
> ```clojure
> (:require ["node:events" :default EventEmitter])
> ```
>
> silently binds `EventEmitter` to `nil`, and `(EventEmitter.)` fails at runtime with `... is not a constructor`. Use `:refer [EventEmitter]`, or `:as` when the module's exports object *is* the thing you want.

---

## Requiring the Closure Library

The Google Closure Library ships with ClojureScript and is required by its namespace as a **symbol**, not a string:

```clojure
(:require [goog.string :as gstring]
          [goog.string.format]           ; loaded only for its side effect
          [goog.object :as gobj])
```

Some Closure namespaces exist only to install a function into another one, which is why `goog.string.format` appears without an alias. Chapter 29 tours the useful parts.

---

## `.cljc`: code for both Clojure and ClojureScript

A `.cljc` file can be loaded by both JVM Clojure and ClojureScript. **Reader conditionals** select platform-specific code:

```clojure
(ns app.shared)

(defn platform []
  #?(:clj  :jvm
     :cljs :javascript))
```

```clojure
(shared/platform)   ;; => :javascript
```

You need `.cljc` for two things: domain code shared with a JVM backend, and **macros**, which always run on the JVM at compile time (chapter 17). Everything in this manual that isn't a macro is `.cljs`.

`#?@` splices a sequence of forms into the surrounding collection: `[1 #?@(:cljs [2 3])]` reads as `[1 2 3]` in ClojureScript.

---

## The REPL and namespaces

At the REPL, `require` is a function. `:reload` picks up changes to a file:

```clojure
(require '[app.util.text :as text] :reload)
(in-ns 'app.util.text)        ; switch the REPL's current namespace
```

Editors do this for you when you load a file. Remember that removing a `def` from the file doesn't remove it from the running program: the old definition stays until you restart.

---

## Rules of thumb

- One namespace per file, path matching the name, with hyphens becoming underscores.
- Require with `:as` and a short, conventional alias. Use `:refer` sparingly.
- Require Node built-ins as strings with the `node:` prefix, and never with `:default`.
- Require Closure namespaces as symbols.
- Use `.cljc` only for code that genuinely has to run on both platforms, and for macros.
- Break cycles by extracting the shared part into its own namespace.
