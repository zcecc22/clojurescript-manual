# 18 — Transducers

*After this chapter you can build reusable transformations that run without intermediate sequences, apply them to collections and reductions, and write your own.*

---

## The idea

A pipeline like `(->> xs (filter odd?) (map inc) (take 3))` builds a lazy sequence at every step and ties each step to its input. A **transducer** is the transformation alone, with no input and no output collection:

```clojure
(def xf (comp (filter odd?) (map inc) (take 3)))
```

`map`, `filter`, `take` and most sequence functions return a transducer when called **without** a collection. `comp` chains them, and you apply the result to data in several ways:

```clojure
(into [] xf (range 100))       ;; => [2 4 6]     into a collection
(sequence xf (range 100))      ;; => (2 4 6)     as a lazy sequence
(transduce xf + (range 100))   ;; => 12          reduce with a function
(transduce xf + 100 (range 100))  ;; => 112      with an initial value
(eduction xf (range 100))      ;; => (2 4 6)     a reducible, recomputed each time it's used
```

The same `xf` works with any source and any destination. It never creates the intermediate sequences, which makes it faster and lighter on memory. On a million elements in a development build, the transducer version of a three-step pipeline ran in 27 ms against 153 ms for the lazy sequence version.

> **Coming from imperative languages:** Think of a transducer as the body of a `for` loop, extracted and made composable: "skip evens, add one, stop after three." It doesn't know whether the items come from a vector, a stream or a database cursor, or whether the results go into a vector, a sum or a socket.

---

## `comp` runs left to right

With functions, `(comp f g)` applies `g` first. With transducers, items flow through in the order you **write** them:

```clojure
(into [] (comp (map inc) (filter even?)) [1 2 3 4])   ;; => [2 4]   inc, then filter
(into [] (comp (filter even?) (map inc)) [1 2 3 4])   ;; => [3 5]   filter, then inc
```

That reads the same as the `->>` pipeline it replaces, so the order is natural. It's only surprising if you know how `comp` treats functions.

---

## Everyday uses

`into` with a transducer is the idiomatic way to transform into a specific collection:

```clojure
(into #{} (map :tag) [{:tag :a} {:tag :b} {:tag :a}])    ;; => #{:a :b}
(into {} (map (juxt :id :name)) [{:id 1 :name "a"}])     ;; => {1 "a"}
```

`transduce` replaces `(reduce f (map g xs))`:

```clojure
(transduce (map :price) + 0 [{:price 3} {:price 4}])   ;; => 7
(transduce (filter pos?) conj [] [-1 2 3])              ;; => [2 3]
```

Transducer versions of common sequence functions:

```clojure
(into [] (partition-all 2) (range 5))   ;; => [[0 1] [2 3] [4]]
(into [] (dedupe) [1 1 2 1])            ;; => [1 2 1]
(into [] (distinct) [1 2 1])            ;; => [1 2]
(into [] (keep identity) [1 nil 2])     ;; => [1 2]
(into [] (mapcat range) [1 2 3])        ;; => [0 0 1 0 1 2]
(into [] (take-while neg?) [-1 -2 3 -4]) ;; => [-1 -2]
(into [] (interpose :sep) [1 2 3])      ;; => [1 :sep 2 :sep 3]
(into [] cat [[1] [2 3]])               ;; => [1 2 3]
(into [] (map-indexed vector) [:a :b])  ;; => [[0 :a] [1 :b]]
```

They also accept JavaScript arrays directly: `(into [] (map inc) (js/Array.from ...))`.

### Eager and exact

`transduce` and `into` are eager, with no chunking (chapter 07). `take` really stops the work:

```clojure
(transduce (comp (map (fn [x] (swap! counter inc) x)) (take 2)) conj [] (range 1000))
;; => [0 1]       and the mapping function ran exactly 2 times
```

The lazy equivalent would have run it 32 times. When a pipeline has side effects or expensive steps, that difference matters.

---

## How a transducer works

A transducer is a function that takes a **reducing function** `rf` and returns a new reducing function. A reducing function has three arities:

- `([])`: initialise, returning an initial accumulator
- `([acc])`: complete, for final cleanup
- `([acc x])`: step, folding one item in

A logging transducer that passes everything through:

```clojure
(defn xf-log [rf]
  (fn
    ([] (rf))
    ([acc] (rf acc))
    ([acc x]
     (println "saw" x)
     (rf acc x))))

(into [] (comp xf-log (filter odd?)) [1 2 3])
;; prints: saw 1, saw 2, saw 3
;; => [1 3]
```

`map` is the same shape with `(rf acc (f x))` in the step. `filter` calls `rf` only when the predicate passes. `take` returns `(reduced acc)` once it has enough, which stops the whole process.

### Stateful transducers

A transducer that needs state creates it when it receives `rf`, which is once per use, so separate runs don't share state. Use a `volatile!` (chapter 11) for it:

```clojure
(defn take-every [n]
  (fn [rf]
    (let [i (volatile! -1)]
      (fn
        ([] (rf))
        ([acc] (rf acc))
        ([acc x]
         (if (zero? (mod (vswap! i inc) n))
           (rf acc x)
           acc))))))

(into [] (take-every 3) (range 10))   ;; => [0 3 6 9]
(into [] (take-every 3) (range 10))   ;; => [0 3 6 9]   fresh state each time
```

> ⚠️ **Gotcha:** Create the state **inside** `(fn [rf] ...)`, never outside it. A volatile created when `take-every` is called would be shared by every use of that transducer, and the second run would start where the first stopped. `partition-all`, `dedupe` and `distinct` are stateful in the same way.

Stateful transducers that buffer, like `partition-all`, use the completion arity to flush what they're holding. Always implement all three arities, and pass completion through.

### `completing`

`transduce` calls the reducing function's completion arity at the end. A plain two-argument function like `+` already has one. For your own, `completing` adds it, optionally with a finishing function:

```clojure
(transduce (map inc) (completing + str) 0 [1 2])   ;; => "5"
```

---

## When to use them

- **Use transducers** for `into`/`transduce` over sizable collections, for any pipeline you apply to several sources, and when you want eager, exact evaluation.
- **Keep `->>` with lazy sequences** for short, one-off transformations at the REPL, for infinite sequences you take from, and when laziness is actually useful.

Converting a hot `->>` pipeline to `(into [] (comp ...) xs)` is one of the cheapest performance wins available (chapter 28).

---

## Rules of thumb

- `(into [] (comp ...) xs)` for collection transforms, `transduce` for reductions.
- Transducers compose left to right, in pipeline order.
- Prefer transducers when a pipeline has side effects or expensive steps. They're eager and exact.
- In a custom transducer, implement all three arities and create state inside `(fn [rf] ...)`.
- Use `volatile!` for transducer state.
