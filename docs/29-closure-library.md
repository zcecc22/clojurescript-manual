# 29 — The Closure Library

*After this chapter you know which parts of the Google Closure Library are worth using from ClojureScript on Node, how to require its namespaces and classes, and which of its functions to avoid.*

---

## What it is

The Closure Library is Google's general-purpose JavaScript library, written for the Closure Compiler and shipped with ClojureScript. `cljs.core` itself is built on it. Because it's compiled together with your code, it's fully optimised: unused functions are removed, so requiring a namespace costs only what you call.

On Node, much of it is redundant with the platform: Node has `fs`, `crypto`, `fetch` and `URL`, and modern JavaScript has most of `goog.array`. What remains useful is a set of small, well-tested utilities.

---

## Requiring it

Namespaces are required as **symbols** (chapter 08). Classes are brought in with `:import`:

```clojure
(ns app.util
  (:require [goog.object :as gobj]
            [goog.string :as gstring]
            [goog.string.format]           ; side effect: installs gstring/format
            [goog.functions :as gfn]
            [goog.crypt :as crypt]
            [goog.math :as gmath])
  (:import [goog.math Long Integer]
           [goog Uri]))
```

> ⚠️ **Gotcha:** A Closure **class** must be `:import`ed. Requiring its namespace with an alias, as in `[goog.i18n.DateTimeFormat :as dtf]`, and then calling `(dtf. ...)` compiles with only an undeclared-var warning and fails at runtime with `... is not a constructor`.

Method calls on Closure class instances produce inference warnings, as with any JS object. The release build worked without hints, because Closure code is renamed consistently with yours, but with `:warnings-as-errors` (chapter 21) you need a hint anyway. `^js` silenced the warnings and the build still worked:

```clojure
(let [^js u (Uri.parse "https://example.com:8443/a/b?y=two")]
  [(.getDomain u) (.getPort u) (.getParameterValue u "y")])
;; => ["example.com" 8443 "two"]
```

---

## `goog.object`: safe JS object access

The workhorse for JS objects with computed keys. String keys are never renamed by advanced compilation (chapter 09):

```clojure
(def o #js {:a 1 :b #js {:c 2}})

(gobj/get o "a")                    ;; => 1
(gobj/get o "zz" :default)          ;; => :default
(gobj/getValueByKeys o "b" "c")     ;; => 2      safe nested access
(vec (gobj/getKeys o))              ;; => ["a" "b"]
(gobj/containsKey o "a")            ;; => true
(gobj/set o "d" 4)                  ; mutates o
(gobj/clone o)                      ; shallow copy
```

`getValueByKeys` returns `undefined` instead of throwing when a level is missing, so it's ClojureScript's `get-in` for JS objects.

---

## `goog.string`

```clojure
(gstring/format "%05d|%-4s|" 42 "ab")   ;; => "00042|ab  |"   needs [goog.string.format]
(gstring/htmlEscape "<a href=\"x\">&</a>")
;; => "&lt;a href=&quot;x&quot;&gt;&amp;&lt;/a&gt;"
(gstring/padNumber 7 3)                 ;; => "007"
(gstring/truncate "hello world" 8)      ;; => "hello..."
(gstring/isEmptyOrWhitespace "  ")      ;; => true
```

`htmlEscape` is the one you'll want if you generate HTML by hand.

---

## `goog.functions`: once, debounce, throttle

```clojure
(let [once-f (gfn/once #(println "init"))
      deb    (gfn/debounce #(swap! calls conj [:debounced %]) 30)
      thr    (gfn/throttle #(swap! calls conj [:throttled %]) 30)]
  (once-f) (once-f)               ; runs once
  (deb 1) (deb 2) (deb 3)         ; one call, 30 ms after the last, with the last args
  (thr 1) (thr 2) (thr 3))        ; first immediately, then at most once per 30 ms
;; calls => [[:throttled 1] [:debounced 3] [:throttled 3]]
```

Debounce suits file-watcher events or config reloads. Throttle suits progress reporting.

---

## Big integers: `goog.math.Long` and `Integer`

ClojureScript numbers lose precision above 2⁵³ (chapter 03). `Long` is an exact signed 64-bit integer, and `Integer` is arbitrary-precision:

```clojure
(let [big (Long.fromString "9007199254740993")]
  (.toString big)                                 ;; => "9007199254740993"
  (.toString (.add big (Long.fromNumber 1)))      ;; => "9007199254740994"
  (.toNumber big))                                ;; => 9007199254740992   lossy, as expected

(.toString (.multiply (Integer.fromString "123456789012345678901234567890")
                      (Integer.fromInt 2)))
;; => "246913578024691357802469135780"
```

JavaScript's built-in `js/BigInt` does the same job and is faster. Use `Long` when you need exactly 64-bit wraparound semantics, such as hashes or IDs from systems with `int64`.

---

## `goog.math`

```clojure
(gmath/clamp 15 0 10)      ;; => 10
(gmath/modulo -7 3)        ;; => 2     like mod
(gmath/toRadians 180)      ;; => 3.141592653589793
```

---

## `goog.crypt`, with a warning

```clojure
(crypt/byteArrayToHex #js [1 255])   ;; => "01ff"
```

> ⚠️ **Gotcha:** `goog.crypt.base64/encodeString` encodes each UTF-16 code unit as a byte (Latin-1), **not UTF-8**:
>
> ```clojure
> (b64/encodeString "héllo")   ;; => "aOlsbG8="
> ```
>
> Every other system expects the UTF-8 encoding, `"aMOpbGxv"`. On Node, use `Buffer`: `(.toString (js/Buffer.from s "utf8") "base64")` (chapter 22).

More generally, don't use `goog.crypt`'s hash implementations for anything security-related. `node:crypto` (chapter 31) is native, faster and maintained for that purpose.

---

## Prefer the platform when it exists

| Need | Use on Node | Not |
|---|---|---|
| URL parsing | `js/URL` | `goog.Uri` (fine, but redundant) |
| Base64 | `Buffer` | `goog.crypt.base64` |
| Hashing, HMAC, random | `node:crypto` | `goog.crypt.*` |
| Date formatting | `Intl.DateTimeFormat` | `goog.i18n.DateTimeFormat` |
| Array helpers | `cljs.core` sequence functions | `goog.array` |

---

## Rules of thumb

- `:require` Closure namespaces as symbols, `:import` Closure classes.
- Use `goog.object` for computed-key access on JS objects.
- Use `goog.string/format` and `htmlEscape`, and `goog.functions` for once, debounce and throttle.
- Use `goog.math.Long` for exact 64-bit integers.
- Hint Closure class instances with `^js` to keep `:warnings-as-errors` builds green.
- Never use `goog.crypt.base64` on text. Use `Buffer`.
- Prefer Node's built-ins where they overlap.
