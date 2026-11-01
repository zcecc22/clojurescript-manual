# 30 — Shipping code

*After this chapter you can package ClojureScript as a CommonJS or ES module that JavaScript code can require or import, design an API boundary JavaScript callers can use, and know what to ship.*

---

## Three ways to ship

| You're shipping | Target | Consumer runs |
|---|---|---|
| A program | `:node-script` | `node out/app.js` or `./out/app.js` |
| A library for CommonJS code | `:node-library` | `require("./mylib.js")` |
| A library for ES-module code | `:esm` | `import { f } from "./mylib.js"` |

Programs were covered in chapters 21 and 26. This chapter is about libraries: ClojureScript code that JavaScript code calls.

---

## The library

```clojure
(ns mylib.core
  (:require [clojure.string :as str]))

(defn slugify
  "Public API: string in, string out."
  [s]
  (-> s str/trim str/lower-case (str/replace #"\s+" "-")))

(defn summarize
  "Public API: takes a JS array of {name, score} objects, returns a JS object."
  [^js people]
  (let [ps     (js->clj people :keywordize-keys true)
        scores (map :score ps)]
    (clj->js {:count (count ps)
              :total (reduce + 0 scores)
              :top   (:name (first (sort-by :score > ps)))})))

(defn fetch-later
  "Public API: returns a Promise."
  [x]
  (js/Promise. (fn [resolve _] (js/setTimeout #(resolve (str "done:" x)) 5))))
```

Notice the boundary discipline in `summarize`: it accepts JavaScript data, converts on the way in, works on ClojureScript data inside, and converts on the way out. JavaScript callers never see a persistent map.

---

## `:node-library`: CommonJS

```clojure
:lib {:target    :node-library
      :output-to "out/mylib.js"
      :exports   {:slugify    mylib.core/slugify
                  :summarize  mylib.core/summarize
                  :fetchLater mylib.core/fetch-later}}
```

`:exports` maps JavaScript names to vars. That's the place to translate `kebab-case` to the `camelCase` JavaScript expects. Only exported functions are reachable after advanced compilation. Everything else is renamed or removed.

```js
const lib = require("./out/mylib.js");

Object.keys(lib);                       // [ 'slugify', 'summarize', 'fetchLater' ]
lib.slugify("  Hello World  ");         // 'hello-world'
lib.summarize([{name: "ada", score: 3}, {name: "bob", score: 9}]);
                                        // { count: 2, total: 12, top: 'bob' }
lib.fetchLater(1).then(console.log);    // 'done:1'
```

The release build is one self-contained file, about 105 KB for this library, most of it the parts of `cljs.core` it uses.

---

## `:esm`: ES modules

```clojure
:esm {:target     :esm
      :output-dir "out/esm"
      :modules    {:mylib {:exports {slugify   mylib.core/slugify
                                     summarize mylib.core/summarize}}}}
```

Each entry in `:modules` becomes a file (`out/esm/mylib.js`) with named exports:

```js
import { slugify, summarize } from "./out/esm/mylib.js";
slugify("ESM  Works");   // 'esm-works'
```

> ⚠️ **Gotcha:** The output is ES-module syntax in a `.js` file. If the nearest `package.json` doesn't say `"type": "module"`, Node 20 prints `[MODULE_TYPELESS_PACKAGE_JSON] Warning: ... Reparsing as ES module`, and every import pays for a failed CommonJS parse first. Ship the ESM output inside a package with `"type": "module"`.

Choose `:esm` when your consumers are modern ES-module code or bundlers, and `:node-library` for CommonJS consumers and maximum compatibility.

---

## Designing the boundary

> ⚠️ **Gotcha:** Returning ClojureScript data to JavaScript hands the caller an opaque object. A function exported as `cljsMap` that returns `{:a 1}` gave the JavaScript caller this:
>
> ```
> ae { A: null, m: 1, j: [ J { fa: null, name: 'a', ... }, 1 ], B: null, l: 16647951, C: 401412 }
> ```
>
> That's a persistent map with names renamed by advanced compilation. JavaScript can't read it, and it changes with every build.

The rules for an API JavaScript can use:

- **Accept and return JS data**: strings, numbers, booleans, arrays and plain objects. Convert at the boundary with `js->clj` and `clj->js`.
- **Return promises for async work**, never callbacks or ClojureScript-specific types.
- **Throw `js/Error`s**, or `ex-info`, which is one. Don't throw maps or keywords (chapter 10).
- **Use camelCase export names and camelCase object keys** in returned data. `clj->js` keeps your keyword names as they are, so `:user-id` becomes `"user-id"`, which JavaScript has to access as `obj["user-id"]`. Rename keys before converting if JavaScript callers will use them.
- **Validate inputs.** JavaScript callers can pass anything. A spec check (chapter 19) at the boundary turns a deep `No protocol method` error into a clear message.

---

## What to ship

- **The release build output only**: `out/mylib.js`, or the `out/esm/` directory. Development builds aren't self-contained (chapter 01).
- **A `package.json`** with `main` (CommonJS) or `exports` and `"type": "module"` (ESM) pointing at the output.
- **Type declarations**, if your consumers use TypeScript: write a small `.d.ts` by hand for the exported functions. Nothing generates one from ClojureScript.

Source maps (chapter 21) are useful for your own debugging, but don't ship them unless you want consumers to see your source.

---

## Rules of thumb

- Use `:node-library` for CommonJS consumers and `:esm` for ES-module consumers.
- Export only the public API, with camelCase names, through `:exports`.
- Accept and return plain JS data. Never let a persistent collection cross the boundary.
- Return promises for async results and throw `Error`s for failures.
- Put ESM output in a package with `"type": "module"`.
- Ship only release output, with a hand-written `.d.ts` if TypeScript users need one.
