# 03 — Values and types

*After this chapter you know what every scalar value really is at runtime, and you can avoid the number and string traps that come from running on JavaScript.*

---

## Types are JavaScript's, plus a few of Clojure's own

Every value has a runtime type, and for scalars it's the JavaScript one:

```clojure
(type 42)      ;; => #object[Number]
(type "s")     ;; => #object[String]
(type true)    ;; => #object[Boolean]
(type #"a+")   ;; => #object[RegExp]
(type nil)     ;; => nil
(type :k)      ;; => cljs.core/Keyword
(type 'sym)    ;; => cljs.core/Symbol
(type [1])     ;; => cljs.core/PersistentVector
(type {:a 1})  ;; => cljs.core/PersistentArrayMap
```

Numbers, strings, booleans and regexes are plain JS values, so they pass to and from JavaScript with no conversion. Keywords, symbols and the collections are ClojureScript types implemented in `cljs.core`.

There are no type declarations. Code is dynamically typed, and you get type safety from tests, spec (chapter 19) and keeping functions small. Type *hints* exist, but only as compiler optimisations (chapter 28).

---

## Numbers are doubles

There is one number type: the IEEE-754 double that JavaScript uses.

```clojure
(+ 0.1 0.2)    ;; => 0.30000000000000004
(/ 10 4)       ;; => 2.5      no integer division, no ratios
(/ 10 5)       ;; => 2
(quot 10 4)    ;; => 2        integer quotient, when you want it
```

There is no `int` vs `float` distinction at runtime. `3.0` and `3` are the same value:

```clojure
(integer? 3.0)   ;; => true
(int? 3.0)       ;; => true
(double? 3)      ;; => true
(= 1 1.0)        ;; => true
```

There's also no overflow error. Past 2⁵³ integers silently lose precision:

```clojure
js/Number.MAX_SAFE_INTEGER     ;; => 9007199254740991
(inc 9007199254740991)         ;; => 9007199254740992
(inc 9007199254740992)         ;; => 9007199254740992   unchanged
```

Division by zero doesn't throw either:

```clojure
(/ 1 0)    ;; => ##Inf
(/ -1 0)   ;; => ##-Inf
(/ 0 0)    ;; => ##NaN
```

> ⚠️ **Gotcha:** Database IDs, Twitter-style snowflake IDs and 64-bit timestamps in nanoseconds can all exceed 2⁵³. If they arrive as JSON numbers they're already corrupted by the time your code sees them. Keep them as strings end to end, or use `js/BigInt`, which is a separate type that doesn't mix with ordinary arithmetic.

> **Coming from imperative languages:** Integer division in Go, Java and C truncates, and Python's `//` floors. Here `/` always gives a fractional result. Use `quot` to truncate toward zero, `rem` for the matching remainder, and `mod` for a remainder that takes the sign of the divisor:
>
> ```clojure
> (rem -7 3)   ;; => -1
> (mod -7 3)   ;; => 2
> ```

### Rounding and conversion

```clojure
(int 2.7)            ;; => 2     truncates toward zero
(int -2.7)           ;; => -2
(Math/round 2.5)     ;; => 3
(js/Math.floor 2.7)  ;; => 2
(.toFixed 3.14159 2) ;; => "3.14"   a string, for display only
```

Parse strings with the core functions, which return `nil` on bad input instead of a half-parsed number:

```clojure
(parse-long "42")      ;; => 42
(parse-long "42px")    ;; => nil
(parse-double "3.5")   ;; => 3.5
(parse-boolean "true") ;; => true

(js/parseInt "42px")   ;; => 42    JS: parses a prefix
(js/parseInt "x")      ;; => ##NaN
```

<!-- CLJS 1.11.51+ --> `parse-long`, `parse-double` and `parse-boolean` are all recent additions. Prefer them to `js/parseInt`, whose prefix parsing hides bad input.

### Bitwise operations are 32-bit

JavaScript bitwise operators work on signed 32-bit integers, so the bit functions do too:

```clojure
(bit-shift-left 1 31)   ;; => -2147483648
(bit-shift-left 1 32)   ;; => 1     the shift count wraps at 32
```

---

## Strings

Strings are JavaScript strings: immutable, UTF-16 internally.

```clojure
(count "abc")        ;; => 3
(subs "hello" 1 3)   ;; => "el"
(get "abc" 1)        ;; => "b"
(seq "ab")           ;; => ("a" "b")
```

There is **no character type**. The literal `\a` reads as the one-character string `"a"`:

```clojure
(type \a)            ;; => #object[String]
(= \a "a")           ;; => true
(str \a \b)          ;; => "ab"
```

> ⚠️ **Gotcha:** `count` counts UTF-16 code units, not characters. An emoji outside the Basic Multilingual Plane is two units:
>
> ```clojure
> (count "😀")                     ;; => 2
> (count (js/Array.from "😀"))     ;; => 1   code points
> ```
>
> Lengths, `subs` and `nth` all work in code units, so slicing user text can split a character in half.

### clojure.string

The string library lives in `clojure.string`, conventionally aliased as `str` (chapter 08 covers `require`):

```clojure
(str/upper-case "ab")          ;; => "AB"
(str/split "a,b,c" #",")       ;; => ["a" "b" "c"]
(str/join "-" [1 2 3])         ;; => "1-2-3"
(str/trim "  x ")              ;; => "x"
(str/includes? "abc" "b")      ;; => true
(str/starts-with? "abc" "a")   ;; => true
(str/replace "a-b-c" "-" "_")  ;; => "a_b_c"   replaces all occurrences
(str/blank? "  ")              ;; => true
```

When `clojure.string` lacks something, call the JavaScript method directly: `(.padStart (str 7) 3 "0")` returns `"007"`. Chapter 09 explains the interop syntax.

### Building and printing strings

`str` concatenates the string forms of its arguments. `nil` becomes the empty string:

```clojure
(str "x=" 1 " " nil :k)    ;; => "x=1 :k"
```

There are two families of printing functions. The `print` family produces human-readable output. The `pr` family produces output the reader can parse back, with strings in quotes:

```clojure
(println "x" 1 nil :k "s")  ;; prints: x 1 nil :k s
(prn "x" 1 nil :k "s")      ;; prints: "x" 1 nil :k "s"
(pr-str {:a "b"})           ;; => "{:a \"b\"}"
```

Use `prn` when debugging, because it shows you the difference between `"1"` and `1` and between `nil` and `"nil"`.

There is no `format` in `cljs.core`. The Closure Library provides one:

```clojure
(ns example.core
  (:require [goog.string :as gstring]
            [goog.string.format]))   ; loads the format function as a side effect

(gstring/format "%s has %d items, %.2f%%" "cart" 3 12.345)
;; => "cart has 3 items, 12.35%"
```

> ⚠️ **Gotcha:** Calling a function that doesn't exist, like `format` written without a namespace, is only a compiler **warning**. The build still succeeds, and the program crashes at runtime with `Cannot read properties of undefined`. Treat every shadow-cljs warning as an error.

---

## `nil` and `undefined`

ClojureScript's `nil` compiles to JavaScript `null`. JavaScript also has `undefined`, which shows up when you read a missing property or call a JS function that returns nothing. Core functions treat the two as the same:

```clojure
(nil? js/undefined)   ;; => true
(= nil js/undefined)  ;; => true
```

So in practice you treat both as `nil`. Most core functions accept `nil` and return something sensible, a habit called *nil punning*:

```clojure
(count nil)     ;; => 0
(first nil)     ;; => nil
(str nil)       ;; => ""
(get nil :a)    ;; => nil
(conj nil 1)    ;; => (1)
```

This removes most `null` checks, but it also means a typo in a key silently gives you `nil` rather than an error, so the mistake surfaces somewhere downstream.

---

## Keywords as functions

Keywords can be called as functions that look themselves up in a map. Maps can be called as functions of their keys:

```clojure
(:a {:a 1})      ;; => 1
({:a 1} :a)      ;; => 1
(:missing {:a 1} :default)  ;; => :default
```

`(:name user)` is the idiomatic way to read a field, and it means you can pass `:name` straight to `map` (chapter 07).

---

## Rules of thumb

- Assume every number is a double. Use `quot`/`rem`/`mod` when you mean integer arithmetic.
- Keep identifiers bigger than 2⁵³ as strings.
- Parse with `parse-long`/`parse-double`, which return `nil` on garbage.
- Debug with `prn`, not `println`, so you can see the types.
- Treat compiler warnings as errors. An undeclared var is a runtime crash waiting to happen.
- Read fields with `(:key m)`.
