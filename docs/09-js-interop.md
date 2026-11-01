# 09 — JS interop

*After this chapter you can call any JavaScript or Node API, move data between ClojureScript and JavaScript deliberately, and avoid the traps where the two worlds disagree.*

---

## The syntax

| You want | Write | JavaScript |
|---|---|---|
| Call a method | `(.toUpperCase s)` | `s.toUpperCase()` |
| Call with arguments | `(.slice s 1 3)` | `s.slice(1, 3)` |
| Read a property | `(.-length s)` | `s.length` |
| Chain | `(.. obj -a -b (method x))` | `obj.a.b.method(x)` |
| A global | `js/process`, `js/Math.PI` | `process`, `Math.PI` |
| Call a global function | `(js/parseInt "42")` | `parseInt("42")` |
| Construct | `(js/Date. 0)` or `(new js/Date 0)` | `new Date(0)` |
| Set a property | `(set! (.-x obj) 10)` | `obj.x = 10` |
| Type check | `(instance? js/Date d)` | `d instanceof Date` |

```clojure
(.toUpperCase "abc")              ;; => "ABC"
(.slice "hello" 1 3)              ;; => "el"
(.-length "abc")                  ;; => 3
(js/Math.max 1 5 3)               ;; => 5
(.. "a-b" (split "-") (join "+")) ;; => "a+b"
(.toISOString (js/Date. 0))       ;; => "1970-01-01T00:00:00.000Z"
```

The dot always comes first: `(.method obj args)` is a call and `(.-prop obj)` is a property read. The `-` is what distinguishes them, and leaving it off calls the property as a function.

Reading a missing property gives `undefined`, which ClojureScript treats as `nil` (chapter 03). Calling a missing method, or reading through a missing object, throws the JavaScript errors you'd expect:

```clojure
(.-nope o)          ;; => nil
(.foo o)            ;; Throws: scratch.core.o.foo is not a function
(.-c (.-zzz o))     ;; Throws: Cannot read properties of undefined (reading 'c')
```

Node's globals are available under `js/`:

```clojure
(.-HOME js/process.env)    ;; => "/home/zcecc22"
(exists? js/window)        ;; => false   safe test for a global that may not exist
```

Module functions come in through `:require` (chapter 08) and are called like any ClojureScript function: `(path/join "a" "b")`.

---

## JavaScript data in ClojureScript

`#js` is a reader literal for JavaScript objects and arrays:

```clojure
(def o #js {:a 1 :b #js {:c 2}})
(.-a o)          ;; => 1
(.. o -b -c)     ;; => 2
(js-keys o)      ;; => #js ["a" "b"]
```

Keyword keys become string property names.

> ⚠️ **Gotcha:** `#js` is **shallow**. In `#js {:d [1 2]}`, the value of `d` is still a ClojureScript vector, not a JS array. Each level needs its own `#js`, or use `clj->js`.

Other constructors: `(js-obj "k" 1)`, `(array 1 2)`, `(into-array [1 2])`.

JavaScript arrays and objects are **mutable**, and there's direct access for them:

```clojure
(let [obj #js {}]
  (set! (.-x obj) 10)
  (aset obj "y" 20)     ;; aset/aget for computed keys (see the note below)
  obj)                  ;; => #js {:x 10, :y 20}

(let [arr #js [1 2 3]]
  (aget arr 1)          ;; => 2
  (count arr)           ;; => 3   core sequence functions accept JS arrays
  (vec arr))            ;; => [1 2 3]
```

`aget` and `aset` are meant for **arrays**. For object properties with computed names, the idiomatic tool is `goog.object`:

```clojure
(:require [goog.object :as gobj])

(gobj/get #js {:a 1} "a")                                ;; => 1
(gobj/getValueByKeys #js {:a #js {:b 2}} "a" "b")        ;; => 2
```

---

## Converting between the worlds

`clj->js` converts deeply: maps become objects, vectors become arrays, keywords become strings:

```clojure
(clj->js {:a 1 :b {:c [1 2 {:d :kw}]}})
;; => #js {:a 1, :b #js {:c #js [1 2 #js {:d "kw"}]}}
```

`js->clj` goes the other way. Without options, keys stay strings:

```clojure
(js->clj #js {"a" 1 "b" #js [1 2]})                 ;; => {"a" 1, "b" [1 2]}
(js->clj #js {"a" 1} :keywordize-keys true)         ;; => {:a 1}
```

The common pattern is to convert once, at the boundary: JSON or a JS library result comes in and becomes ClojureScript data, your code works on that, and the result goes back out through `clj->js`.

```clojure
(js->clj (js/JSON.parse "{\"user_id\": 1, \"tags\": [\"a\"]}") :keywordize-keys true)
;; => {:user_id 1, :tags ["a"]}
```

> ⚠️ **Gotcha:** `clj->js` drops keyword namespaces: `{:ns/key 1}` becomes `#js {:key 1}`, so `:user/id` and `:order/id` collide. Rename namespaced keys before converting.

> ⚠️ **Gotcha:** `js->clj` only converts plain objects and arrays. Class instances, like a `Date`, a `Buffer` or a stream, pass through unchanged, which is usually what you want. Converting large payloads both ways has a real cost (chapter 23).

---

## `this`

ClojureScript functions don't use `this`. When a JavaScript API calls your function as a method and you need the receiver, `this-as` binds it:

```clojure
(def counter
  #js {:n 0
       :inc (fn []
              (this-as this
                (set! (.-n this) (inc (.-n this)))
                (.-n this)))})

(.inc counter)   ;; => 1
(.inc counter)   ;; => 2
```

> ⚠️ **Gotcha:** As in JavaScript, pulling a method off its object loses `this`. `(let [f (.-inc counter)] (f))` doesn't throw: it runs with the wrong receiver and returns `##NaN`. Call methods with `(.method obj)`, or bind them with `(.bind f obj)`.

You'll need `this-as` rarely. Most Node APIs pass what you need as arguments.

---

## Printing JavaScript and ClojureScript values

`prn` prints JS objects readably, as `#js {...}`. `js/console.log` prints ClojureScript data badly:

```clojure
(js/console.log "console:" #js {:a 1} {:b 2})
;; prints: console: { a: 1 } { meta: null, cnt: 1, arr: [ ... ], __hash: null, ... }
```

That's the internal structure of a persistent map. Use `prn` for ClojureScript data and keep `console.log` for JS values, or convert first with `clj->js`.

---

## JavaScript semantics leak through

Arithmetic compiles to JavaScript operators, so type mixing coerces silently at runtime:

```clojure
(inc "1")     ;; => "11"
(+ 1 "2")     ;; => "12"
(* "3" 2)     ;; => 6
(+ 1 nil)     ;; => 1
```

The compiler warns when it can see the problem, as with these literal strings. Values arriving at runtime from JSON or user input get no check at all.

> ⚠️ **Gotcha:** A number that arrives as a string, like a query parameter, an environment variable or a CSV field, will concatenate instead of add. Parse at the boundary with `parse-long` or `parse-double` (chapter 03).

---

## Advanced compilation and property names

In a release build the Closure Compiler renames properties to save space. That's safe for ClojureScript code, but a JavaScript object's property names have to survive, or `(.-userName user)` would read a renamed property that doesn't exist.

shadow-cljs handles this with **externs inference**: it tracks which values are JavaScript and preserves the property names you access on them. Usually it works without help. In a verified release build, `(.-userName user)` on a parsed JSON object kept its name, with no annotations.

When shadow-cljs can't tell what a value is, it says so:

```
------ WARNING #1 - :infer-warning ------
 Cannot infer target type in expression (. (Readable.from x) -readableObjectMode)
```

**That warning means the release build is broken.** In that build `.-readableObjectMode` was renamed to `.Tb`. The development build ran correctly and the release build read `undefined`. The fix is a `^js` hint on the binding, which tells the compiler the value is a JavaScript object:

```clojure
(defn mode [x]
  (let [^js r (.from Readable x)]
    (.-readableObjectMode r)))

(defn user-name [^js user]
  (.-userName user))
```

> ⚠️ **Gotcha:** One renaming bug produced **no warning at all**. Calling a static method on a `:refer`'d class with the dotted-symbol form, `(Readable.from xs)`, compiled cleanly and failed in the release build with `Cannot read properties of undefined (reading 'from')`, because `Readable` itself was renamed. These forms all survived the same release build:
>
> ```clojure
> (.from Readable xs)             ; method call on the referred class
> (stream/Readable.from xs)       ; dotted access through an :as alias
> (EventEmitter.)                 ; constructing a referred class
> ```
>
> Use `(.method Class args)` for static methods, and test the `release` build, not only the development build.

`goog.object/get` with a string key is always safe, because string keys are never renamed. Chapter 21 covers release builds in detail.

---

## Rules of thumb

- `(.method obj)` calls, `(.-prop obj)` reads. Don't forget the `-`.
- Convert at the boundary: `js->clj` with `:keywordize-keys` on the way in, `clj->js` on the way out.
- Remember `#js` is shallow. Use `clj->js` for nested data.
- Use `goog.object` for computed property names, `aget`/`aset` only for arrays.
- Use `prn` for ClojureScript values, never `console.log`.
- Parse strings to numbers at the edge. Arithmetic won't complain.
- Fix every externs inference warning with a `^js` hint. It means the release build is broken.
- Call static methods as `(.method Class args)`, and run the release build before shipping.
