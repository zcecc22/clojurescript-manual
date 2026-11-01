# ClojureScript Style Guide

Rules, not preferences. Each one is decidable: given a piece of code, you can say whether it follows the rule.

The ordering is by how often it comes up in review. If you only internalise the first four sections you will write code that looks like it belongs.

---

## 1. Formatting

There's no `gofmt` in core ClojureScript, but there is a near-universal convention, and editors with Clojure support apply most of it automatically. (The community tool for checking it is `cljfmt`, outside this manual's core-only scope.)

**Closing parens gather on the last line.** Never put a closing paren on a line of its own.

```clojure
;; Good
(defn total [items]
  (reduce + 0 (map :price items)))
```

```clojure
;; Bad — C-style closing
(defn total [items]
  (reduce + 0 (map :price items))
)
```

**Bodies indent two spaces.** This applies to forms with a body (`defn`, `let`, `when`, `fn`, `cond`, `doseq` and the like).

**Function-call arguments align, or indent one space.** If the arguments don't fit on one line, put each on its own line, aligned with the first argument:

```clojure
;; Good
(assoc-in db [:users id]
          {:name name :email email})

;; Good — first argument on the next line, one-space indent
(str
 "a long string "
 "continued")
```

**Bindings and map entries align in pairs.** In `let`, `cond`, `case` and multi-line maps, keep each key and value on one line. Aligning the values vertically is common and optional.

```clojure
(let [user  (find-user id)
      roles (:roles user)]
  ...)
```

**No line-length religion,** but around 80 characters is the community norm, and long lines usually mean a form wants a `let` or a threading macro.

**One blank line between top-level forms.** None inside a function body. If a function needs blank lines to be readable, it wants splitting.

---

## 2. Naming

### kebab-case, always

`max-retry-count`, `parse-user`, `user-id`. Never `camelCase` or `snake_case` in ClojureScript names. Use camelCase only where you're forced to talk to JavaScript: export names (chapter 30) and JS object keys.

### Suffixes and prefixes carry meaning

| Pattern | Means | Example |
|---|---|---|
| `name?` | returns a boolean | `valid?`, `empty?`, `admin?` |
| `name!` | has side effects or mutates | `save!`, `reset-cache!`, `swap!` |
| `a->b` | converts a to b | `user->json`, `->request` |
| `*name*` | dynamic var (chapter 11) | `*level*` |
| `-name` or `defn-` | private, internal | `defn- normalize*` |
| `name*` | a variant or helper of `name` | `normalize*` |
| `_` or `_name` | deliberately unused | `(fn [_ x] x)` |

```clojure
;; Good
(defn active? [user] (= :active (:status user)))
(defn save-user! [db user] ...)

;; Bad — predicate without ?, side effect without !
(defn is-active [user] ...)
(defn save-user [db user] ...)
```

### No `get-` prefix

A function that computes a value is named for the value: `(user-roles user)`, not `(get-user-roles user)`. Keep `get-` for the rare function that actually fetches something remotely, and even then prefer `fetch-` or `load-`.

### Length scales with scope

Short names for short scopes: `x`, `m`, `k`, `v`, `acc`, `coll` and `xs` in a three-line function are clearer than long ones. Top-level functions get full names.

Conventional short names, worth using exactly:

| Name | For |
|---|---|
| `m` | a map |
| `coll` | any collection |
| `xs`, `ys` | a sequence |
| `k`, `v` | key and value |
| `f`, `g` | functions |
| `n` | a count or number |
| `acc` | a reduce accumulator |
| `e` | a caught error |
| `req`, `res` | request and response |

### Keywords are namespaced when they cross a boundary

Map keys used within one namespace can be plain (`:id`). Keys in data that travels, such as events, messages and specs, are namespaced (`:user/id`, `:order/id`) so they can't collide.

---

## 3. Namespaces

**One namespace per file, path matching the name** (chapter 08).

**Require with `:as` and a short alias. `:refer` sparingly.** A qualified call says where a function comes from.

```clojure
;; Good
(ns app.orders
  (:require [clojure.string :as str]
            [app.db :as db]
            ["node:path" :as path]))

(db/insert! conn (str/trim name))
```

```clojure
;; Bad — where do insert! and trim come from?
(ns app.orders
  (:require [clojure.string :refer [trim]]
            [app.db :refer [insert!]]))
```

`:refer` is fine for a handful of names used constantly: `deftest`, `is` and `testing` in test namespaces, and `EventEmitter`-style JS classes.

**Use the conventional aliases.** `str`, `set`, `walk`, `edn`, `pprint`, `s` (spec), `gobj` and `gstring` are recognised on sight. Don't invent `cs` for `clojure.string`.

**Order requires:** ClojureScript namespaces, then JS strings. Sort alphabetically within each group.

**No `utils`, `helpers` or `common` namespaces.** Name a namespace for what it's about: `app.money`, `app.text`, `app.http-client`.

---

## 4. Data and functions

**Model data as plain maps.** Reach for `defrecord` only when protocol dispatch earns it (chapter 15).

**Functions take data and return data.** Keep side effects in a thin outer layer, and keep the logic pure and testable.

**More than three or four parameters: take a map.**

```clojure
;; Good
(defn create-user [{:keys [name email role] :or {role :user}}] ...)
(create-user {:name "Ada" :email "a@b.co"})

;; Bad
(defn create-user [name email role active? notify?] ...)
(create-user "Ada" "a@b.co" :user true false)
```

**Always use the return value.** A call to `assoc`, `conj` or `update` whose result is discarded is a bug.

**Prefer `->`/`->>` to deep nesting.** Three or more nested calls want a thread.

```clojure
;; Good
(->> orders
     (filter :paid?)
     (map :total)
     (reduce + 0))

;; Bad
(reduce + 0 (map :total (filter :paid? orders)))
```

**`->` for collections, `->>` for sequences** (chapter 04). Don't mix positions within one thread. Use `as->` or a `let` instead.

**Short anonymous functions only.** `#(...)` for one short expression, `fn` with named parameters otherwise. Never nest `#()`.

**Destructure in the parameter list** when you use more than one field.

---

## 5. Control flow

**`when` for one-armed conditionals, `if` for two arms.** Never write `(if test x nil)`.

**`cond` for chains, with `:else` as the last test.**

**`case` only for literal constants, with a default** unless an unmatched value should crash (chapter 05).

**`if-let`/`when-let` instead of `let` followed by `if`:**

```clojure
;; Good
(when-let [user (find-user id)]
  (greet user))

;; Bad
(let [user (find-user id)]
  (when user
    (greet user)))
```

**`seq` to test for non-empty**, never `(not (empty? x))`:

```clojure
(when (seq xs) ...)       ; good
(when (not (empty? xs)) ...)  ; bad
```

**`doseq` for side effects, never `map` or `for`.**

---

## 6. State

**One atom per area of state, holding a map**, defined with `defonce` (chapter 11).

**`swap!` with a named pure function**, not a long anonymous one:

```clojure
;; Good
(defn add-item [cart item] (update cart :items conj item))
(swap! state update :cart add-item item)

;; Bad
(swap! state (fn [s] ... twenty lines ...))
```

**Never `deref` an atom twice when one value is meant.** Read it once into a local.

```clojure
;; Bad — two reads may see different states across an await
(when (:ready? @state) (process (:queue @state)))

;; Good
(let [{:keys [ready? queue]} @state]
  (when ready? (process queue)))
```

**No `def` inside functions.** `def` always creates a namespace-level var. Use `let`.

---

## 7. Interop

**Convert at the boundary:** `js->clj` on the way in, `clj->js` on the way out, plain ClojureScript data in between (chapter 09).

**Hint JS values with `^js`** whenever you read their properties, especially values that come from `await` (chapter 22). Zero inference warnings.

**`(.method Class args)` for static methods,** not `(Class.method args)` on a `:refer`'d name (chapter 09).

**`prn` for ClojureScript values, never `js/console.log`.**

**Require Node built-ins with the `node:` prefix and never `:default`** (chapter 08).

---

## 8. Async

**`^:async`/`await` for anything with more than one step.** `.then` for single transforms.

**Every promise chain ends in a handler.** An async function's result is either awaited or `.catch`ed (chapter 12).

**No `await` inside nested `fn`s.** Use `doseq`/`loop`, or map to promises and `Promise.all` them.

**Every network call has a timeout.**

**Bound concurrency** for unbounded inputs (chapter 14).

---

## 9. Errors

**Throw `ex-info` with a `:type` keyword** in the data map, and the inputs that caused the failure (chapter 10).

```clojure
;; Good
(throw (ex-info "insufficient funds" {:type :insufficient-funds :balance b :amount a}))

;; Bad
(throw (js/Error. (str "insufficient funds: " b " < " a)))
(throw :insufficient-funds)
```

**Error messages are lowercase phrases without trailing punctuation,** so they compose when wrapped: `"config unreadable: unexpected token"`.

**Catch narrowly, rethrow what you don't handle.**

**Wrap with a cause:** `(ex-info msg data cause)`, never a new error that drops the original.

---

## 10. Comments and documentation

**Docstrings on every public function,** before the parameter vector. The first line is a sentence that stands alone:

```clojure
(defn slugify
  "Returns s lower-cased, trimmed, with runs of whitespace replaced by '-'."
  [s]
  ...)
```

**Comment conventions by semicolon count:**

| | Used for |
|---|---|
| `;` | end-of-line remarks |
| `;;` | comments on their own line, indented with the code |
| `;;;` | section headings at the top level |

**Comment *why*, not *what*.** The code says what.

**Keep REPL experiments in a `(comment ...)` block** at the end of the file. That's the conventional place for them, and it's fine to commit.

**`#_` is for temporarily disabling a form**, never for permanent dead code. Delete it.

---

## 11. Testing style

**Test namespaces end in `-test` and mirror the source namespace:** `app.cart` is tested by `app.cart-test` (chapter 20).

**`(is (= expected actual))`**, comparing whole values.

**`are` for tables**, `testing` for labelled groups.

**Async tests use `async` with both branches handled and `done` in `.finally`.**

---

## 12. Review checklist

- [ ] No closing parens on their own line, and bodies indent two spaces
- [ ] Names are kebab-case, with `?` on predicates and `!` on side effects
- [ ] Requires use `:as` aliases, `:refer` only for a few test or interop names
- [ ] No discarded return value from `assoc`/`conj`/`update`/`assoc!`
- [ ] Domain data is plain maps, and functions over four parameters take a map
- [ ] `when` for one arm, `seq` for non-empty, `doseq` for side effects
- [ ] State lives in `defonce` atoms, updated with named pure functions
- [ ] JS values read with `.-prop` are hinted `^js`, and the build has zero warnings
- [ ] Every promise chain ends in a handler, and every network call has a timeout
- [ ] Errors are `ex-info` with a `:type`, and causes are preserved
- [ ] Public functions have docstrings, and REPL experiments are in `(comment ...)`

---

## Sources

This guide condenses the community conventions, reordered by frequency and adapted to ClojureScript on Node:

- [The Clojure Style Guide](https://guide.clojure.style/)
- [How to ns](https://stuartsierra.com/2016/clojure-how-to-ns.html), for namespace declarations
- The idioms of `cljs.core` itself

Where this manual's chapters found a ClojureScript- or shadow-cljs-specific trap, the rule here follows the chapter.
