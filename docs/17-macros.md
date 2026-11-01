# 17 — Macros

*After this chapter you can read and write macros, set up macro namespaces the way shadow-cljs expects, avoid the classic hygiene bugs, and recognise when a function would do.*

---

## What a macro is

A macro is a function that runs **at compile time**. It receives the unevaluated code of its arguments as data (lists, symbols, vectors) and returns new code, which the compiler then compiles in place of the macro call.

Because code is data (chapter 02), a macro builds code with the same functions you use on any data. That's how `when`, `cond`, `->`, `defn` and most of the language's convenience forms are implemented:

```clojure
(macroexpand '(when x y))   ;; => (if x (do y))
```

> **Coming from imperative languages:** This isn't the C preprocessor's text substitution, and it isn't reflection or code generation in a build step. A macro operates on the parsed program, with the whole language available, and its output is checked like any other code. Go's answer to the same need is `go generate`. Lisp makes it part of compilation.

---

## Macros run on the JVM

The ClojureScript compiler is written in Clojure and runs on the JVM, so **macros are Clojure, not ClojureScript**. They live in a `.clj` or `.cljc` file. The usual setup is a pair of files with the same namespace name:

`src/app/macros.clj`, holding the macros:

```clojure
(ns app.macros)

(defmacro unless [test & body]
  `(if ~test nil (do ~@body)))
```

`src/app/macros.cljs`, holding runtime code, which also pulls in its own macros:

```clojure
(ns app.macros
  (:require-macros [app.macros]))

(defn helper [x] (str "runtime helper: " x))
```

With the `:require-macros` line, consumers require the namespace once and get both macros and functions through the same alias:

```clojure
(ns app.main
  (:require [app.macros :as m :refer [unless]]))

(unless false :ran)   ;; => :ran
(unless true :ran)    ;; => nil
(m/helper 1)          ;; => "runtime helper: 1"
```

A single `.cljc` file can do both jobs. The reader conditional makes only the ClojureScript side require its own macros:

```clojure
;; src/app/dsl.cljc
(ns app.dsl
  #?(:cljs (:require-macros [app.dsl])))

(defmacro defroutes [name & pairs]
  `(def ~name ~(into {} (map vec (partition 2 pairs)))))
```

```clojure
(defroutes routes "/" :home "/about" :about)
routes   ;; => {"/" :home, "/about" :about}
```

The macro body runs on the JVM, so it can call Clojure functions (`partition`, `into`) freely, but not JavaScript. The code it *returns* runs in Node, so that code can use `js/` interop.

---

## Syntax-quote

Macros mostly build code with **syntax-quote**, the backtick:

| Form | Meaning |
|---|---|
| `` `(...) `` | template: return this code, don't evaluate it |
| `~x` | unquote: insert the value of `x` here |
| `~@xs` | unquote-splicing: insert the elements of `xs` here |
| `name#` | auto-gensym: a unique local name, the same within one template |

```clojure
`(a b ~(+ 1 2) ~@[4 5])
;; => (scratch.core/a scratch.core/b 3 4 5)

`map     ;; => cljs.core/map
```

Syntax-quote **namespace-qualifies every symbol**. `map` becomes `cljs.core/map` and an unknown `a` becomes `current-ns/a`. That's deliberate: a macro's output refers to the functions the macro author meant, even if the caller has a local named `map`.

`macroexpand-1` shows one step of expansion, which is the main debugging tool:

```clojure
(macroexpand-1 '(unless false :ran))   ;; => (if false nil (do :ran))
```

---

## Hygiene

### Evaluate arguments exactly once

A macro argument is code. Inserting it twice runs it twice:

```clojure
(defmacro square-bad [x]
  `(* ~x ~x))

(defn noisy [] (println "evaluated!") 3)

(m/square-bad (noisy))
;; prints "evaluated!" twice  => 9
```

Bind the argument to a local once, using an auto-gensym so it can't collide with the caller's names:

```clojure
(defmacro square-good [x]
  `(let [v# ~x] (* v# v#)))

(m/square-good (noisy))
;; prints "evaluated!" once  => 9
```

### Locals need gensyms

Because syntax-quote qualifies symbols, a plain local name in a template doesn't compile:

```clojure
(defmacro broken-let [v & body]
  `(let [x ~v] ~@body))

(m/broken-let 1 :x)
;; Does not compile: Invalid local name: app.macros/x
```

Use `x#`. The error is the compiler protecting you from accidentally capturing a caller's `x`.

### Deliberate capture

Occasionally you *want* to introduce a name the caller can use. `~'it` inserts the unqualified symbol:

```clojure
(defmacro with-captured [& body]
  `(let [~'it 42] ~@body))

(m/with-captured (inc it))   ;; => 43
```

These "anaphoric" macros are surprising to readers, who see `it` with no binding. Use them rarely, and document them.

### A practical macro

Timing a block needs a macro, because the block must run *inside* the timing code:

```clojure
(defmacro timed [label & body]
  `(let [start#  (js/performance.now)
         result# (do ~@body)]
     (println ~label "took" (.toFixed (- (js/performance.now) start#) 1) "ms")
     result#))

(timed "sum" (reduce + (range 100000)))
;; prints: sum took 4.3 ms
;; => 4999950000
```

---

## Macros aren't values

A macro exists only at compile time. It can't be passed to `map`, stored or applied.

> ⚠️ **Gotcha:** Using a macro as a value isn't caught at compile time in this setup. `(map unless [true false])` compiled without an error and crashed at runtime with `Cannot read properties of undefined (reading 'cljs$core$IFn$_invoke$arity$1')`. If a macro needs to be passed around, wrap it in a function: `#(unless % :x)`.

---

## When not to write a macro

The first rule of macros is to not write one when a function will do. A function composes, can be passed around, shows up in stack traces, and can be redefined at the REPL. A macro has none of those properties.

Write a macro only when you need to:

- **Control evaluation:** evaluate an argument conditionally, repeatedly, or inside some context. `unless` and `timed` are examples.
- **Introduce bindings:** `let`-like or `with-`-style forms.
- **Do work at compile time:** compute a value or a lookup table from literal arguments, as `defroutes` does.
- **Remove boilerplate that a function can't**, such as a family of `def`s.

When you do write one, keep it thin: have the macro do the minimum syntax work and delegate to a function that does the real work. The function is testable and redefinable, and the macro stays trivial.

---

## Rules of thumb

- Prefer a function. Write a macro only for evaluation control, bindings or compile-time work.
- Put macros in `.clj`, or `.cljc`, with a `.cljs` namespace that `:require-macros` itself.
- Use syntax-quote. Every local in a template is `name#`.
- Evaluate each argument exactly once.
- Check expansions with `macroexpand-1`.
- Keep macros thin and delegate to functions.
- Remember macros aren't values. Wrap one in a function to pass it around.
