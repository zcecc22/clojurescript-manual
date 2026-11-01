# 02 — Syntax and evaluation

*After this chapter you can read any ClojureScript form, know what it evaluates to, and predict how truthiness and equality behave.*

---

## Forms

ClojureScript code is made of **forms**. The reader turns text into data structures, and the compiler evaluates that data. That's the whole syntax:

| Form | Example | Evaluates to |
|---|---|---|
| Number | `42`, `3.14`, `1e3` | itself |
| String | `"hello"` | itself |
| Keyword | `:name`, `:user/id` | itself |
| Symbol | `greet`, `str/join` | the value it names |
| Vector | `[1 2 3]` | a vector of its evaluated elements |
| Map | `{:a 1 :b 2}` | a map of its evaluated keys and values |
| Set | `#{1 2 3}` | a set of its evaluated elements |
| List | `(f a b)` | **a call**: `f` applied to `a` and `b` |
| `nil`, `true`, `false` | | themselves |
| Regex | `#"a+"` | a JS `RegExp` |

A list is the one form that isn't data by default: the first element is the operator and the rest are its arguments. `(+ 1 2)` calls `+`. `(if c a b)` is a *special form* that the compiler handles itself. Function calls, macros and special forms all share this shape.

```clojure
(+ 1 2)          ;; => 3
'(+ 1 2)         ;; => (+ 1 2)    quoted: a list, not a call
(list '+ 1 2)    ;; => (+ 1 2)
(first '(+ 1 2)) ;; => +          the symbol itself
```

Quoting with `'` stops evaluation, which leaves you holding the data the reader produced. That code is data is what makes macros possible (chapter 17), but day to day you'll quote rarely, mostly in `require` at the REPL.

> **Coming from imperative languages:** There is no operator precedence because there are no infix operators. `(+ 1 (* 2 3))` is unambiguous. `+`, `<` and `=` are ordinary functions and take any number of arguments: `(+ 1 2 3 4)`, `(< 1 2 3)` checks that the arguments are ascending.

---

## Whitespace and commas

Elements are separated by whitespace. **Commas are whitespace too.** `{:a 1, :b 2}` and `{:a 1 :b 2}` are identical. People use commas in maps occasionally for readability and almost nowhere else.

Comments:

```clojure
; a line comment (by convention ;; for whole-line comments)

[1 #_2 3]   ;; => [1 3]   #_ discards the next form, whatever its size

(comment
  (this is never evaluated))
```

`#_` is the one you'll use constantly: it comments out a single form without having to find its closing paren.

`(comment ...)` is a macro that ignores its body and returns `nil`. It's the conventional home for REPL experiments at the bottom of a file. The code inside has to read correctly (balanced parens) but is never compiled, and you evaluate pieces of it from your editor.

---

## Symbols and keywords

These look similar and do completely different jobs.

A **symbol** is a name that refers to something: `greet`, `str/join`, `+`. When evaluated, it's replaced by the value it names. A namespaced symbol like `str/join` refers to `join` in whatever namespace was aliased as `str`.

A **keyword** is a value that stands for itself: `:name`, `:status`. Keywords are what you use for map keys, enum-like values and option flags:

```clojure
{:status :active, :role :admin}
```

Keywords can be namespaced to avoid collisions: `:user/id` is a different keyword from `:order/id`. `::local` auto-namespaces to the current namespace:

```clojure
:user/id               ;; => :user/id
::local                ;; => :scratch.core/local   (inside ns scratch.core)
(name ::local)         ;; => "local"
(namespace ::local)    ;; => "scratch.core"
```

> **Coming from imperative languages:** Keywords replace string constants, enums and struct field names all at once. Where Go has `type Status int` with `iota` constants, ClojureScript just writes `:pending` and `:active`.

> ⚠️ **Gotcha:** Keywords are **not interned** in ClojureScript. Each `:a` can be a separate object, so `(identical? :a :a)` returns `false`. Compare keywords with `=` (or `keyword-identical?`), never `identical?`. JVM Clojure behaves differently, so code ported from it can break here.

---

## Truthiness: only `nil` and `false`

```clojure
(map boolean [nil false 0 "" [] {} () js/NaN])
;; => (false false true true true true true true)
```

Exactly two values are falsy: `nil` and `false`. Everything else is truthy, **including `0`, `""`, empty collections and `NaN`**.

```clojure
(if nil "yes" "no")    ;; => "no"
(not 0)                ;; => false
```

This is a deliberate break from the JavaScript runtime underneath. To test for emptiness, ask directly:

```clojure
(empty? [])            ;; => true
(seq [])               ;; => nil    the idiomatic "is there anything?" test
(when-not (seq []) :empty)  ;; => :empty
```

`and` and `or` return one of their *arguments*, not a boolean. `and` returns the first falsy value or the last value. `or` returns the first truthy value or the last value:

```clojure
(and 1 2 3)          ;; => 3
(and 1 nil 3)        ;; => nil
(or nil false :x)    ;; => :x
(or nil false)       ;; => false
```

`(or x default)` is the standard way to supply a default.

> ⚠️ **Gotcha:** JavaScript habits say `0` and `""` are falsy, and Python habits say empty lists are. Neither is true here. `(if (count xs) ...)` is always true, because `0` is truthy. Write `(if (seq xs) ...)` or `(if (pos? (count xs)) ...)`.

---

## Equality

`=` is **value equality**, and it's the one you want almost every time:

```clojure
(= {:a 1 :b 2} {:b 2 :a 1})   ;; => true    maps: key order doesn't matter
(= [1 2 3] '(1 2 3))          ;; => true    sequential collections compare by contents
(= [1 2] #{1 2})              ;; => false   a vector is not a set
(= "abc" (str "ab" "c"))      ;; => true
```

Vectors and lists are equal if they hold equal elements in the same order. That makes `=` deep: nested collections compare by content all the way down.

`identical?` is **reference equality**, JavaScript's `===`. You rarely need it, and its results for immutable values depend on compiler details (the empty vector is a shared constant, for instance), so don't build logic on it.

JavaScript objects and arrays are mutable, and `=` treats them by reference:

```clojure
(= #js [1] #js [1])           ;; => false
(= (js-obj) (js-obj))         ;; => false
```

> ⚠️ **Gotcha:** `(= 1 1.0)` is `true`, because they're the same JavaScript number (chapter 03). `(= "1" 1)` is `false`: there's no type coercion. `##NaN` is not `=` to itself. Test for it with `NaN?`, as in `(NaN? x)`.

`compare` gives ordering: a negative number, zero or a positive number. It works on numbers, strings, keywords and vectors, and it's what `sort` uses:

```clojure
(compare "a" "b")   ;; => -1
(compare 2 1)       ;; => 1
```

---

## `do` and side effects

A function body or `let` body can hold several forms. All are evaluated, and the value of the last is returned. Elsewhere, `do` groups forms explicitly:

```clojure
(do (println "side effect") 42)
;; prints: side effect
;; => 42
```

Needing `do` is a hint that you're doing side effects. That's fine at the edges of a program, but a pure function never needs it.

---

## Rules of thumb

- Read `(f a b)` as "call `f`". Anything else in the code is data that evaluates to itself.
- Use keywords for map keys and enumerated values, and strings for text from the outside world.
- Test emptiness with `seq` or `empty?`, never with truthiness of a count.
- Use `=` for comparison. Reach for `identical?` only when you mean reference identity on a mutable JS object.
- Comment out a form with `#_`. Keep REPL experiments in a `(comment ...)` block.
