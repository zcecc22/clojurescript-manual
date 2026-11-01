# 12 — Promises and async

*After this chapter you can consume and create promises, write sequential-looking async code with `^:async` and `await`, run work in parallel, and make sure no rejection goes unhandled.*

---

## Promises are just JavaScript

There is no separate async library in core ClojureScript. Node's asynchronous APIs return JavaScript promises, and you use them through ordinary interop:

```clojure
(defn sleep [ms]
  (js/Promise. (fn [resolve _reject] (js/setTimeout resolve ms))))

(defn fetch-user [id]
  (js/Promise.
    (fn [resolve reject]
      (js/setTimeout
        #(if (pos? id)
           (resolve {:id id :name (str "user" id)})
           (reject (ex-info "bad id" {:id id})))
        10))))
```

A promise can resolve to any value, ClojureScript maps included. Reject with an `ex-info` so the error carries data (chapter 10).

---

## `^:async` and `await`

<!-- CLJS 1.12+ --> Mark a function `^:async` and you can `await` promises inside it, exactly as in JavaScript:

```clojure
(defn ^:async demo []
  (let [u (await (fetch-user 1))]
    (prn :step1 u)
    (let [caught (try
                   (await (fetch-user -1))
                   (catch :default e [:caught (ex-message e) (ex-data e)]))]
      (prn caught)
      :demo-done)))

(.then (demo) #(prn :result %))
;; prints:
;; :step1 {:id 1, :name "user1"}
;; [:caught "bad id" {:id -1}]
;; :result :demo-done
```

An async function always returns a promise, whatever its body returns. `try`/`catch`/`finally` work across `await`: a rejected promise throws at the `await`.

`await` works inside `let`, `do`, `if`, `try`, `doseq` and `loop`/`recur`:

```clojure
(defn ^:async ticks []
  (doseq [i [1 2]]
    (await (sleep 5))
    (prn :doseq i))
  (loop [i 0]
    (when (< i 2)
      (await (sleep 5))
      (prn :loop i)
      (recur (inc i)))))
```

For an anonymous async function, name it, or put the metadata on `fn`:

```clojure
(fn ^:async load [] (await (sleep 1)) :done)   ; works
(^:async fn [] (await (sleep 1)) :done)        ; also works
(fn ^:async [] ...)                            ; Does not compile: await can only be used in async contexts
```

> ⚠️ **Gotcha:** `await` only works directly in the async function's own body. Inside a nested `fn`, including the one you pass to `map`, `mapv` or `reduce`, it's a compile error:
>
> ```clojure
> ;; Does not compile: Assert failed: await can only be used in async contexts
> (defn ^:async g [xs]
>   (mapv (fn [x] (await (js/Promise.resolve x))) xs))
> ```
>
> Iterate with `doseq` or `loop` for sequential awaits, or map to promises and `await` them all together (see below).

> **Coming from imperative languages:** This is JavaScript's `async`/`await` with the same semantics. It isn't Go's goroutines: the code doesn't run in parallel. Each `await` yields the single thread until the promise settles. Python's `asyncio` is the closer model.

---

## `.then` chains

Without `async`, you chain with `.then` and `.catch`. A `.then` callback that returns a promise is flattened, so the next step gets the resolved value:

```clojure
(-> (fetch-user 9)
    (.then (fn [u] (prn :then u) (fetch-user 10)))
    (.then (fn [u] (prn :flattened u) (throw (ex-info "oops" {:at :then}))))
    (.catch (fn [e] (prn :catch (ex-message e) (ex-data e)) :recovered))
    (.then (fn [v] (prn :after-catch v)))
    (.finally (fn [] (prn :finally))))
;; prints:
;; :then {:id 9, :name "user9"}
;; :flattened {:id 10, :name "user10"}
;; :catch "oops" {:at :then}
;; :after-catch :recovered
;; :finally
```

A throw inside any `.then` rejects the chain. `.catch` handles it and its return value continues the chain. The `->` macro makes the chain read top to bottom.

Prefer `^:async` for anything with more than two steps or any branching. Keep `.then` for one-step transforms and the top-level call site.

---

## Running promises together

Starting a promise starts the work. Running things in parallel means starting them all, then waiting for them all:

```clojure
(defn ^:async parallel []
  (let [both (await (js/Promise.all #js [(fetch-user 2) (fetch-user 3)]))]
    (vec both)))
;; => [{:id 2, :name "user2"} {:id 3, :name "user3"}]
```

The combinators take a JavaScript array (or other iterable) of promises:

| Combinator | Resolves when | Rejects when |
|---|---|---|
| `js/Promise.all` | all fulfil, to an array of values | any rejects |
| `js/Promise.allSettled` | all settle, to an array of `{status, value/reason}` | never |
| `js/Promise.race` | the first settles | the first settles with a rejection |
| `js/Promise.any` | the first fulfils | all reject |

```clojure
(mapv #(.-status %) (await (js/Promise.allSettled #js [(fetch-user 1) (fetch-user -1)])))
;; => ["fulfilled" "rejected"]
```

The map-then-wait pattern replaces `await` inside `map`:

```clojure
(await (js/Promise.all (clj->js (map fetch-user [4 5]))))
;; => #js [{:id 4, :name "user4"} {:id 5, :name "user5"}]
```

`clj->js` turns the lazy sequence into an array (and forces it, which starts all the work). `into-array` does the same. The result is a JS array of ClojureScript maps. Use `vec` on it if you need a vector.

> ⚠️ **Gotcha:** `(map fetch-user ids)` is lazy, so nothing starts until something realises the sequence. Pass it straight to `into-array` or `clj->js`, and don't hold onto the lazy sequence expecting the work to be running.

`Promise.all` is unbounded: a thousand IDs start a thousand requests at once. Chapter 14 shows how to limit concurrency.

---

## Callback APIs

Most Node modules have a promise version, so prefer `node:fs/promises` over `node:fs`. Where only a callback version exists, `node:util`'s `promisify` converts it:

```clojure
(:require ["node:util" :as util]
          ["node:fs" :as fs])

(def read-file (util/promisify fs/readFile))

(.then (read-file "shadow-cljs.edn" "utf8") #(prn (subs % 0 14)))
;; prints: "{:source-paths"
```

---

## Unhandled rejections

A rejected promise that nothing handles **crashes the process**. Node prints the error and exits with code 1, even if other timers are pending:

```clojure
(js/setTimeout #(js/Promise.reject (ex-info "nobody catches me" {:x 1})) 20)
(js/setTimeout #(prn :never-printed) 200)
;; prints: Error: nobody catches me   (and the stack), then exits with code 1
```

Every promise chain needs an end that handles errors: a `.catch` on the outermost call, or a `try` around the `await` in the outermost async function. Calling an async function and ignoring its result is fire-and-forget, and if it fails it takes the process down.

```clojure
(defn ^:async run []
  (let [u (await (fetch-user -1))]
    (prn u)))

(defn main [& _]
  (-> (run)
      (.catch (fn [e]
                (prn :fatal (ex-message e) (ex-data e))
                (set! (.-exitCode js/process) 1)))))
```

> ⚠️ **Gotcha:** `try` around a call to an async function **without** `await` catches nothing: the call returns a promise immediately and the rejection happens later. `(try (demo) (catch ...))` is always a bug. Write `(try (await (demo)) ...)` inside an async function, or `.catch` on the promise.

---

## Rules of thumb

- Use `^:async` and `await` for sequential async logic, and `.then` for one-step transforms.
- Reject with `ex-info`, so errors carry data.
- Never `await` inside a nested `fn`. Use `doseq`/`loop`, or map to promises and `Promise.all` them.
- Start work in parallel, then await together.
- Every chain ends in a handler. An unhandled rejection kills the process.
- `try` only catches an async error if there's an `await` inside it.
