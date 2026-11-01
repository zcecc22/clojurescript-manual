# 28 — Performance

*After this chapter you know what's expensive in ClojureScript, can measure it properly, and have a short list of changes that fix most hot spots.*

---

## Measure first, and measure release

Persistent data structures and laziness have a real cost, but most code isn't hot, and idiomatic ClojureScript is fast enough for the large majority of it. Optimise what you've measured.

A minimal benchmark harness that warms up, then averages:

```clojure
(defn bench [label n f]
  (dotimes [_ 3] (f))                       ; warm up the JIT
  (let [t0 (js/performance.now)]
    (dotimes [_ n] (f))
    (println (str label ": " (.toFixed (/ (- (js/performance.now) t0) n) 2) " ms"))))
```

All the numbers below come from a `shadow-cljs release` build on Node 20.19, on one machine. Treat them as ratios, not absolutes. For this kind of tight code, development builds were within about 20% of release. The difference is larger for code that leans on `cljs.core` dispatch.

For real profiling, `node --cpu-prof out/app.js` writes a `.cpuprofile` you can open in Chrome DevTools. Use a `release --debug` build (chapter 21) so the function names are readable.

---

## Laziness costs; transducers and loops don't

Over a vector of 1,000,000 numbers, "increment, keep evens, sum":

| Version | Time |
|---|---|
| `(->> xs (map inc) (filter even?) (reduce +))` | 101.6 ms |
| `(transduce (comp (map inc) (filter even?)) + xs)` | 15.1 ms |
| `loop`/`recur` with `nth` | 9.3 ms |

The lazy pipeline allocates a lazy sequence, with chunks and cons cells, at every step. The transducer allocates nothing in between. Converting a hot `->>` pipeline to `transduce` or `into` (chapter 18) is the single cheapest large win available, and it keeps the code idiomatic. Drop to `loop`/`recur` only if the remaining gap matters.

---

## Transients for building collections

Building a collection one element at a time with `conj` or `assoc` creates a new persistent collection per step. **Transients** are mutable versions for use inside a single function:

```clojure
(persistent! (reduce conj! (transient []) (range 1000000)))
```

| Building | Persistent | Transient |
|---|---|---|
| vector of 1e6 with `conj` | 48.7 ms | 12.6 ms |
| map of 1e5 with `assoc` | 48.2 ms | 7.9 ms |
| `(into [] (range 1000000))` | 12.3 ms (uses transients internally) | |
| JS array `.push` 1e6 | 11.5 ms | |

`into`, `mapv`, `filterv`, `vec` and `set` already use transients, so the idiomatic forms are fast. Write transients by hand only in a hot loop that builds a collection step by step.

The rules: create with `transient`, change with the `!` functions (`conj!`, `assoc!`, `dissoc!`, `disj!`, `pop!`), finish with `persistent!`, and never let the transient escape the function.

> ⚠️ **Gotcha:** Like the persistent functions, `assoc!` and `conj!` **return** the updated collection, and you must use the return value. "Bashing in place", calling `assoc!` and ignoring the result, silently loses data:
>
> ```clojure
> (let [t (transient {})]
>   (dotimes [i 20] (assoc! t i i))   ; result ignored
>   (count (persistent! t)))          ;; => 8, not 20
> ```
>
> It works for the first eight entries and then the map switches representation (chapter 06), and the rest goes into a new object that was thrown away. Thread the value through `reduce` or `loop`.

Using a transient after `persistent!` throws: `conj! after persistent!`.

---

## The JSON boundary

Parsing 20,000 small records:

| | Time |
|---|---|
| `js/JSON.parse` | 7.5 ms |
| `JSON.parse` plus `js->clj :keywordize-keys` | 58.5 ms |

Conversion is about 8× the cost of parsing. For a typical API payload that's microseconds and irrelevant. For large payloads you only pass through, filter, or read a few fields from, skip `js->clj`: work on the JS objects with interop, or convert only the part you keep (`(js->clj (.-items parsed))`).

---

## Map lookup vs JS properties

One million field reads:

| | Time |
|---|---|
| `(:b m)` on a small map | 4.5 ms |
| `(.-b o)` on a JS object | 0.2 ms |

A keyword lookup goes through protocol dispatch, while a JS property read is a single machine-level load that the JIT can often hoist out of the loop entirely. 4.5 ns per lookup is still fast. This matters only in the innermost loops of numeric or parsing code, where a JS object or array is a legitimate optimisation.

---

## Strings

| Building a 100,000-char string | Time |
|---|---|
| `(recur ... (str s "x"))` in a loop | 0.7 ms |
| `.join` on a JS array | 1.6 ms |
| `(apply str (repeat 100000 "x"))` | 4.8 ms |

Repeated concatenation is fast in V8, which builds ropes rather than copying. `apply str` over a lazy sequence pays for the sequence. For large text, collect parts in a JS array and `.join` them, or concatenate directly.

---

## Type hints

`^number` and `^boolean` hints let the compiler emit plain JavaScript operators instead of generic checks in some places:

```clojure
(defn sum-sq [^number a ^number b]
  (+ (* a a) (* b b)))
```

They matter much less than in JVM Clojure, since arithmetic already compiles to JS operators, and wrong hints silently produce wrong code. The `^js` hint (chapter 09) is the one you'll use constantly, and it's for correctness under advanced compilation, not speed.

---

## The checklist

In rough order of payoff:

1. **Algorithmic fixes.** A nested `filter` inside `map` is O(n²). Build a map or set index with `group-by` or `into #{}` first.
2. **Lazy pipelines on hot paths** become `transduce` or `into` with transducers.
3. **Repeated `js->clj`** of large payloads: convert less, or not at all.
4. **Step-by-step collection building** in loops: transients, or `into`.
5. **Blocking the event loop**: chunk the work, or move it to a worker (chapter 14).
6. **Innermost numeric loops**: JS arrays, `aget`/`aset`, `loop`/`recur`.

---

## Rules of thumb

- Benchmark the release build, warmed up, with `performance.now`.
- Replace lazy `->>` with transducers on hot paths.
- Let `into`, `mapv` and `vec` use transients for you. Hand-written transients must thread the return value.
- Convert big JSON only as far as you need.
- Use JS arrays and objects only in measured inner loops.
- Profile with `--cpu-prof` on a `--debug` release build.
