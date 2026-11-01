# 07 — Sequences

*After this chapter you can express any loop as a pipeline of sequence functions, and you know where laziness will surprise you.*

---

## The seq abstraction

Every collection, plus strings, `nil` and many JavaScript iterables, can be viewed as a **sequence**: something with a `first` and a `rest`. `seq` produces that view, or `nil` when there's nothing there:

```clojure
(seq [1 2])     ;; => (1 2)
(seq {:a 1})    ;; => ([:a 1])
(seq "ab")      ;; => ("a" "b")
(seq [])        ;; => nil
(seq nil)       ;; => nil
```

The sequence functions (`map`, `filter`, `reduce` and around a hundred others) all call `seq` on their input. Write one function against sequences and it works on vectors, lists, sets, maps and strings.

```clojure
(first [1 2])     ;; => 1
(rest [1])        ;; => ()     always a sequence, possibly empty
(next [1])        ;; => nil    nil when empty
(cons 0 [1 2])    ;; => (0 1 2)
```

> **Coming from imperative languages:** This replaces the iterator protocols you know: Go's `range`, Python's `__iter__`, Java's `Iterable`. The difference is that the vocabulary on top of it is huge and uniform, so most loops become one line.

---

## The core pipeline

```clojure
(map inc [1 2 3])             ;; => (2 3 4)
(filter odd? (range 10))      ;; => (1 3 5 7 9)
(remove odd? (range 6))       ;; => (0 2 4)
(reduce + [1 2 3])            ;; => 6
(reduce + 10 [1 2 3])         ;; => 16    with an initial value
(reduce + [])                 ;; => 0     (+) with no args is 0
```

`map` takes several sequences and stops at the shortest:

```clojure
(map vector [1 2 3] [:a :b])  ;; => ([1 :a] [2 :b])
(map-indexed vector [:a :b])  ;; => ([0 :a] [1 :b])
```

Common relatives:

```clojure
(keep #(when (pos? %) (* 10 %)) [-1 2 -3 4])  ;; => (20 40)   map, then drop nils
(mapcat (fn [x] [x x]) [1 2])                 ;; => (1 1 2 2)  map, then concatenate
```

Always pass an initial value to `reduce` unless the function handles every case. With an empty collection and no initial value, `reduce` calls the function with no arguments.

### Working with records of data

A sequence of maps is the most common shape of real data:

```clojure
(def people [{:name "Ada" :dept :eng :age 36}
             {:name "Bob" :dept :ops :age 25}
             {:name "Cy"  :dept :eng :age 41}])

(map :name people)                  ;; => ("Ada" "Bob" "Cy")
(map :name (sort-by :age > people)) ;; => ("Cy" "Ada" "Bob")
(group-by :dept people)
;; => {:eng [{:name "Ada", ...} {:name "Cy", ...}], :ops [{:name "Bob", ...}]}
(apply max-key :age people)         ;; => {:name "Cy", :dept :eng, :age 41}
(into {} (map (juxt :name :age)) people)
;; => {"Ada" 36, "Bob" 25, "Cy" 41}
```

Keywords being functions is what makes these one-liners: `:name` is the field extractor and `:age` is the sort key.

### Slicing, grouping and ordering

```clojure
(take 3 (range))                    ;; => (0 1 2)
(drop 2 [1 2 3 4])                  ;; => (3 4)
(take-while neg? [-2 -1 0 1])       ;; => (-2 -1)
(drop-while neg? [-2 -1 0 1])       ;; => (0 1)
(partition 2 [1 2 3 4 5])           ;; => ((1 2) (3 4))       drops the incomplete tail
(partition-all 2 [1 2 3 4 5])       ;; => ((1 2) (3 4) (5))
(partition-by odd? [1 3 2 4 5])     ;; => ((1 3) (2 4) (5))
(frequencies [:a :b :a])            ;; => {:a 2, :b 1}
(sort > [3 1 2])                    ;; => (3 2 1)
(distinct [1 2 1 3])                ;; => (1 2 3)
(dedupe [1 1 2 1])                  ;; => (1 2 1)              consecutive duplicates only
```

> ⚠️ **Gotcha:** `partition` silently drops a trailing group that's too short. That's almost never what you want for batching. Use `partition-all`.

### Predicates over sequences

```clojure
(some even? [1 3 4])       ;; => true   first truthy result, or nil
(every? pos? [1 2])        ;; => true
(not-any? neg? [1 2])      ;; => true
(not-empty [])             ;; => nil    the collection itself, or nil
```

### Transforming maps

```clojure
(update-vals {:a 1 :b 2} inc)     ;; => {:a 2, :b 3}       CLJS 1.11+
(update-keys {:a 1} name)         ;; => {"a" 1}
(reduce-kv (fn [acc k v] (assoc acc v k)) {} {:a 1 :b 2})
;; => {1 :a, 2 :b}
```

---

## Results are sequences, not vectors

`map`, `filter` and friends return **lazy sequences**, whatever went in:

```clojure
(vector? (map inc [1]))     ;; => false
(= (map inc [1 2]) [2 3])   ;; => true    equality still works
```

When you need a vector, because you'll index into it, `conj` onto the end, or want eager evaluation, use `mapv`/`filterv` or `into`:

```clojure
(mapv inc [1 2 3])           ;; => [2 3 4]
(into [] (map inc) [1 2])    ;; => [2 3]
(into #{} [1 1 2])           ;; => #{1 2}
(into '() [1 2 3])           ;; => (3 2 1)   conj onto a list reverses
```

`(into [] (map inc) coll)`, with `map` given no collection, is a transducer (chapter 18). It skips building the intermediate lazy sequence.

---

## `for`: sequence comprehensions

`for` builds a lazy sequence from nested bindings, with filters and local bindings:

```clojure
(for [x [1 2 3]
      y [:a :b]
      :when (odd? x)
      :let  [z (str x y)]]
  z)
;; => ("1:a" "1:b" "3:a" "3:b")

(for [x (range 10) :while (< x 3)] x)   ;; => (0 1 2)
```

It's Python's list comprehension, nested loops included. It is **not** a loop: it produces a value and does nothing until that value is consumed.

---

## Laziness

Lazy sequences compute elements only when something asks for them. That's what makes infinite sequences usable:

```clojure
(take 5 (iterate #(* 2 %) 1))   ;; => (1 2 4 8 16)
(take 4 (cycle [:a :b]))        ;; => (:a :b :a :b)
(first (filter #(> % 10) (range)))  ;; => 11
```

It also has costs, and they're the source of most sequence bugs.

### Side effects inside lazy sequences

```clojure
(def side (map (fn [x] (println "realising" x) x) (range 100)))
(println "defined")
(first side)
```

Prints `defined` first, and only then the `realising` lines, because the `map` doesn't run until `first` asks for an element. And when it does run, it prints **32** lines, not one:

> ⚠️ **Gotcha:** Sequences over vectors and ranges are **chunked**: they're realised 32 elements at a time. Asking for the first element of `(map f (range 100))` calls `f` 32 times. So laziness never guarantees "only as much as needed": it guarantees "at least as much as needed, possibly more".

Never put side effects inside `map`, `filter` or `for`. For effects, use:

```clojure
(run! prn [:a :b])          ;; calls prn on each, returns nil
(doseq [x xs] (prn x))      ;; same, with destructuring and nesting
(doall (map f xs))          ;; force a lazy seq and keep the result
(dorun (map f xs))          ;; force it and discard the result
```

### Laziness and dynamic scope

A lazy sequence created inside `try`, `binding` or `with-redefs` may be realised after that scope has closed. An exception thrown while realising it then escapes the `try` that was meant to catch it. If a lazy computation can throw, realise it inside the `try`, with `doall` or `vec`.

---

## Rules of thumb

- Think in pipelines: `->>` with `map`, `filter`, `reduce`.
- Use `mapv`, `filterv` or `into []` when you need a vector or eager evaluation.
- No side effects in lazy sequence functions. Use `run!` or `doseq`.
- Expect chunking: a lazy sequence over a vector or range realises 32 elements at a time.
- Always give `reduce` an initial value.
- Use `partition-all` for batching, not `partition`.
