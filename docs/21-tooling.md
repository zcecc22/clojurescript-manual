# 21 — Tooling

*After this chapter you can configure shadow-cljs builds for development and release, inject configuration at compile time, keep advanced compilation from breaking your interop, and debug release output.*

---

## Anatomy of `shadow-cljs.edn`

```clojure
{:source-paths ["src" "test"]
 :dependencies []

 :builds
 {:app  {:target    :node-script
         :main      app.main/main
         :output-to "out/app.js"
         :compiler-options {:warnings-as-errors true}
         :release   {:compiler-options {:source-map true}}}

  :test {:target    :node-test
         :output-to "out/test.js"
         :ns-regexp "-test$"}}}
```

- **`:source-paths`**: where namespaces are found. Changing it requires restarting a running `shadow-cljs server` (chapter 20).
- **`:dependencies`**: Clojure libraries as `[group/artifact "version"]`. npm packages go in `package.json` instead.
- **`:builds`**: named builds. Each has a `:target` and its own options.
- **`:release`** and **`:dev`**: options merged into the build only in that mode.

The Node targets:

| Target | Produces | Use for |
|---|---|---|
| `:node-script` | one runnable script, calling `:main` with the arguments | CLIs, servers, workers |
| `:node-library` | a CommonJS module with chosen exports | code called from JavaScript (chapter 30) |
| `:esm` | ES modules | modern JS consumers (chapter 30) |
| `:node-test` | a test runner script | tests (chapter 20) |

A project can have as many builds as it needs: an app, a worker script (chapter 14) and tests, all sharing `src/`.

---

## Development and release

| | `compile` / `watch` | `release` |
|---|---|---|
| Optimisation | none | Closure `:advanced` |
| Output | small loader plus files in `.shadow-cljs/` | one self-contained file |
| Names | readable | renamed (`Be.la`) |
| `goog.DEBUG` | `true` | `false` |
| Asserts, `:pre`/`:post` | enabled | removed by default |
| Hot reload, REPL | yes | no |

Advanced compilation removes unused code across the whole program, including the parts of `cljs.core` you don't use, and renames everything it can. The hello-world from chapter 01 releases to about 90 KB.

---

## Treat warnings as errors

Several mistakes in this manual are only **warnings** in ClojureScript: calling an undeclared function (chapter 03), the wrong arity (chapter 04), an inference failure (below). The build succeeds and the program fails later. Turn them into errors:

```clojure
:compiler-options {:warnings-as-errors true}
```

```
------ ERROR -------------------------------------------------------------------
Use of undeclared Var tool.core/no-such-fn
```

The build now fails, with exit code 1, which is exactly what CI needs.

---

## Compile-time configuration with `goog-define`

`goog-define` declares a constant whose value can be set per build. In a release build it's inlined, and dead branches are removed:

```clojure
(ns tool.core)

(goog-define API_URL "http://localhost:8080")
(goog-define VERBOSE false)

(defn main [& _]
  (prn :api API_URL :verbose VERBOSE :debug? goog.DEBUG))
```

```clojure
:tool {:target :node-script
       :main tool.core/main
       :output-to "out/tool.js"
       :closure-defines {tool.core/VERBOSE true}
       :release {:closure-defines {tool.core/API_URL "https://api.example.com"}}}
```

```
compile:  :api "http://localhost:8080" :verbose true :debug? true
release:  :api "https://api.example.com" :verbose true :debug? false
```

The value must be a string, number or boolean. A define in the config with no matching `goog-define` in the code produces an `unknown @define variable` warning, which catches typos and stale config.

For anything that varies per *deployment* rather than per build, such as secrets, hostnames or ports, read `js/process.env` at runtime instead (chapter 26). Compile-time defines are for build flavours.

---

## Externs inference

Chapter 09 introduced the problem: advanced compilation renames properties, so property access on JavaScript objects must be preserved. shadow-cljs infers which accesses are on JS values, and **warns when it can't tell**:

```
------ WARNING #1 - :infer-warning ---------------------------------------------
 Cannot infer target type in expression (. (Readable.from x) -readableObjectMode)
```

This warning is not cosmetic. In the release build of that code, `.-readableObjectMode` was renamed to `.Tb` and the program read `undefined`, while the development build worked fine. A `^js` hint fixes it:

```clojure
(defn mode [x]
  (let [^js r (.from Readable x)]
    (.-readableObjectMode r)))
```

With `:warnings-as-errors` on, an inference warning fails the build, so it can't slip through.

> ⚠️ **Gotcha:** Not every renaming problem warns. `(Readable.from xs)`, a static method on a `:refer`'d class written as a dotted symbol, compiled with zero warnings and broke in release because `Readable` itself was renamed. `(.from Readable xs)` worked. Run your test suite or a smoke test against the **release** build, not just the development build.

---

## Debugging release builds

When something works in development and fails in release, it's almost always renaming. Two tools help:

**Pseudo-names.** `--debug` keeps advanced compilation but uses readable names, so a stack trace says `tool$core$main` instead of `Be`:

```sh
npx shadow-cljs release tool --debug
```

**Source maps.** `:compiler-options {:source-map true}` in `:release` writes `out/tool.js.map` next to the output.

> ⚠️ **Gotcha:** In this setup, `node --enable-source-maps out/tool.js` did **not** translate release stack traces: they still pointed at `tool.js:218:735`. The map was valid, and querying it with Node's `module.SourceMap` API resolved positions to `.cljs` files, but shadow-cljs writes an *index* map (with `sections`) and Node 20 didn't apply it. Keep the map for offline decoding, and rely on `--debug` for readable traces.

Development builds don't have this problem. Their stack traces already point into the original sources, as in `.../cljs-runtime/app/main.cljs:8:11`.

---

## Hot reload on Node

`watch` recompiles on save and reloads the changed namespaces into the running process. Code that holds resources needs to cooperate: `defonce` state (chapter 11), and functions tagged with lifecycle metadata to stop and restart things around a reload:

```clojure
(ns tool.core)

(defonce state (atom 0))
(defn greeting [] "v1")

(defn ^:dev/before-load stop []
  (println "before-load"))

(defn ^:dev/after-load start []
  (println "after-load, greeting now" (greeting) "state" @state))

(defn main [& _]
  (swap! state inc)
  (println "main" (greeting))
  (js/setInterval (fn []) 1000))     ; keep the process alive
```

With `npx shadow-cljs watch tool` running, start `node out/tool.js`, then change `"v1"` to `"v2"` and save:

```
main v1
shadow-cljs - #7 ready!
before-load
after-load, greeting now v2 state 1
```

The new definition is live, `main` didn't run again, and the `defonce` atom kept its value. In a real server, `stop` closes the listener held in a `defonce` atom and `start` opens a new one.

Without these, a reload redefines your handler functions, but a server started at load time keeps calling the old ones, or a second server tries to bind the same port.

---

## Reader conditionals and `.cljc`

Code shared with JVM Clojure lives in `.cljc` files, with `#?(:clj ... :cljs ...)` choosing per platform (chapter 08). The tooling consequence is that `.cljc` code is compiled twice, once by each platform's compiler, so both branches must at least read correctly. Keep platform-specific code in small leaf functions, not scattered through shared logic.

---

## Rules of thumb

- Turn on `:warnings-as-errors`. Every ClojureScript warning is a latent runtime bug.
- Every `:infer-warning` means the release build is broken. Fix it with `^js`.
- Test the release build. Development success proves nothing about renaming.
- Use `goog-define` for build flavours and `process.env` for deployment config.
- Debug release problems with `--debug` first.
- Pair `watch` with `defonce` and `^:dev/before-load`/`^:dev/after-load` for anything holding a resource.
