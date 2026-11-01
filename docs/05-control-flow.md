# 05 — Control flow

*After this chapter you can branch, bind and iterate in expression style, and you know what to write when your instinct says `return` or `for (i = 0; ...)`.*

---

## Everything returns a value

`if` is an expression. It returns the value of whichever branch ran:

```clojure
(if (> 3 2) "yes" "no")   ;; => "yes"
(if false "yes")          ;; => nil      a missing else branch gives nil
```

There is no ternary operator because `if` already is one. This matters more than it looks: since every form returns a value, you compute results directly instead of declaring a variable and assigning to it in each branch.

> **Coming from imperative languages:** The pattern
>
> ```go
> var label string
> if n < 0 { label = "negative" } else { label = "non-negative" }
> ```
>
> becomes `(let [label (if (neg? n) "negative" "non-negative")] ...)`. No variable is ever assigned twice.

`if` takes exactly one form per branch. For several, use `do`, or `when` when there's no else branch:

```clojure
(when true
  (println "w")
  :ok)            ;; prints: w   => :ok

(when false :x)   ;; => nil
```

Use `when` when you only care about the true case, especially when it has side effects. `when-not` and `if-not` are the negated versions.

---

## `cond`: if/else-if chains

```clojure
(defn classify [n]
  (cond
    (neg? n)  :negative
    (zero? n) :zero
    (< n 10)  :small
    :else     :large))

(map classify [-1 0 5 50])   ;; => (:negative :zero :small :large)
```

Clauses are test/result pairs, tried in order. `:else` is just a truthy keyword that catches everything else, a convention rather than syntax. Without a matching clause, `cond` returns `nil`.

`condp` is for when every test applies the same predicate to the same value:

```clojure
(condp = 2
  1 :one
  2 :two
  :other)        ;; => :two
```

---

## `case`: constant dispatch

```clojure
(defn http-text [code]
  (case code
    200           "OK"
    404           "Not Found"
    (500 502 503) "Server Error"
    "Unknown"))                    ; default: a lone trailing expression

(map http-text [200 404 503 418])
;; => ("OK" "Not Found" "Server Error" "Unknown")
```

`case` matches against literal constants (numbers, strings, keywords, vectors) and compiles to a fast lookup. A list groups several constants into one clause.

> ⚠️ **Gotcha:** Without a default, an unmatched value throws instead of returning `nil`:
>
> ```clojure
> (case 7 1 :one)   ;; Throws: No matching clause: 7
> ```

> ⚠️ **Gotcha:** The test values are **literals**, never evaluated. A symbol in a test position is matched as the symbol, not as the value it names:
>
> ```clojure
> (def ONE 1)
> (case 1
>   ONE :matched-symbol
>   1   :matched-1)       ;; => :matched-1
> ```
>
> If you need to match against named constants, use `condp =` or `cond`.

---

## Binding and testing in one step

`if-let` and `when-let` bind a value and branch on whether it's truthy:

```clojure
(if-let [x (seq [])]
  x
  :none)                   ;; => :none

(when-let [u {:n 1}]
  (:n u))                  ;; => 1
```

`if-some` and `when-some` test for non-`nil` instead of truthy, so `false` counts as a value:

```clojure
(if-some [x false] [:some x] :none)   ;; => [:some false]
(if-let  [x false] [:some x] :none)   ;; => :none
```

---

## `let`

`let` binds names to values for the extent of its body. Later bindings can use earlier ones, and a name can be rebound. That creates a new binding, it doesn't mutate the old one:

```clojure
(let [x 1
      x (inc x)
      y (* x 10)]
  [x y])         ;; => [2 20]
```

Prefer a few well-named `let` bindings to one deeply nested expression. Destructuring (chapter 04) works in every `let`.

---

## Life without early return

There's no `return`, `break` or `continue`. What replaces them depends on what you were doing:

**Guard clauses** become a `cond`, with the edge cases first:

```clojure
(defn price [{:keys [qty unit]}]
  (cond
    (nil? qty)   0
    (neg? qty)   (throw (ex-info "negative qty" {:qty qty}))
    :else        (* qty unit)))
```

**"Find the first match and stop"** is `some`, or `first` with `filter`. Both stop early, `filter` because it's lazy:

```clojure
(some #(when (> % 2) %) [1 2 3 4])          ;; => 3
(first (filter #(> % 10) (range)))           ;; => 11    (range) is infinite
```

**"Accumulate, but stop at some point"** is `reduce` with `reduced`, which ends the reduction immediately:

```clojure
(reduce (fn [acc x]
          (if (> x 2) (reduced acc) (+ acc x)))
        0 [1 2 3 4])                         ;; => 3
```

**Validation that can fail** usually means throwing (chapter 10) or returning `nil` and letting the caller decide.

> **Coming from imperative languages:** Go's `if err != nil { return err }` and Python's guard-clause style both lean on early exit. Here the function is a single expression, so its shape *is* its control flow. Deep nesting is the warning sign that a function wants splitting, the same as in any language.

---

## Iteration

In everyday code you rarely write a loop. You transform sequences with `map`, `filter` and `reduce` (chapter 07). For the cases that remain:

### `loop`/`recur`

`loop` sets up bindings like `let`. `recur` jumps back to the top of the loop with new values:

```clojure
(loop [i 0, acc []]
  (if (< i 3)
    (recur (inc i) (conj acc i))
    acc))                          ;; => [0 1 2]
```

`recur` must be in **tail position**: the last thing evaluated on that path. The compiler checks this:

```clojure
;; Does not compile: Can't recur here
(defn bad [n]
  (if (zero? n) 0 (+ 1 (recur (dec n)))))
```

`recur` without a `loop` jumps to the top of the enclosing function, as in chapter 04.

### `doseq` and `dotimes`: for side effects

`doseq` iterates for side effects and always returns `nil`. Several bindings nest, the first one outermost:

```clojure
(doseq [x [1 2]
        y [:a :b]]
  (println x y))
;; prints:
;; 1 :a
;; 1 :b
;; 2 :a
;; 2 :b

(dotimes [i 3] (println i))   ;; prints 0, 1, 2
```

`while` exists, but it's only useful with mutable state:

```clojure
(let [i (atom 0)]
  (while (< @i 3) (swap! i inc))
  @i)                           ;; => 3
```

> ⚠️ **Gotcha:** On Node, every call to `print` or `println` goes out as its own `console.log`, so **each `print` call ends up on its own line**. `(dotimes [i 3] (print i))` prints three lines, not `012`. To build one line, assemble the string first: `(println (apply str (range 3)))`.

> ⚠️ **Gotcha:** `for` is not a loop. It's a lazy sequence comprehension (chapter 07). Writing `(for [x xs] (println x))` for side effects prints nothing unless something consumes the result, and in a REPL it will appear to work because the REPL consumes it to print it. Use `doseq` for side effects.

---

## Rules of thumb

- Compute values with `if`/`cond`/`case`. Never declare a name just to assign it in branches.
- Use `when` for one-armed conditionals and `when-let` to bind and test together.
- `case` only for literal constants, and always give it a default unless an unmatched value should crash.
- Replace "loop and break" with `some`, `filter` plus `first`, or `reduce` with `reduced`.
- Use `doseq` for side effects, `map` and friends for values, and `loop`/`recur` only when neither fits.
