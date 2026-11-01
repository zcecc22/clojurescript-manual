# 13 — Events and streams

*After this chapter you can work with Node's `EventEmitter`, consume async iterables without `for await`, and process data with streams and pipelines.*

---

## `EventEmitter`

Much of Node is built on `EventEmitter`: servers, sockets, child processes and streams all emit named events. Require the class with `:refer` (chapter 08 explains why not `:default`):

```clojure
(ns app.events
  (:require ["node:events" :as events :refer [EventEmitter]]))

(def bus (EventEmitter.))

(defn on-order [order] (prn :order order))

(.on bus "order" on-order)
(.once bus "order" (fn [o] (prn :first-only o)))

(.emit bus "order" {:id 1})
;; prints:
;; :order {:id 1}
;; :first-only {:id 1}
;; => true                 emit returns whether anyone was listening

(.emit bus "order" {:id 2})   ;; prints: :order {:id 2}   (the once-listener is gone)

(.off bus "order" on-order)
(.listenerCount bus "order")  ;; => 0
```

Payloads can be any value, ClojureScript maps included, and several arguments arrive as several parameters:

```clojure
(.on bus "multi" (fn [a b] (prn :multi a b)))
(.emit bus "multi" 1 2)       ;; prints: :multi 1 2
```

To remove a listener with `.off`, you need the same function object you registered. Keep a reference, as `on-order` above. An inline `fn` can't be removed.

Listeners run **synchronously**, in registration order, inside `.emit`. A slow listener blocks the emitter and everything after it.

### The `"error"` event

> ⚠️ **Gotcha:** `"error"` is special. Emitting it with no listener registered **throws** the error:
>
> ```clojure
> (.emit bus "error" (ex-info "unhandled error event" {}))
> ;; Throws: unhandled error event
> ```
>
> For a stream, socket or server, that throw happens inside Node's internals and crashes the process. Every emitter you create or receive that can fail needs an `"error"` listener.

```clojure
(.on bus "error" (fn [e] (prn :error-handled (ex-message e))))
```

### Waiting for one event

`events/once` returns a promise of the event's arguments, as a JS array:

```clojure
(defn ^:async wait-ready []
  (js/setTimeout #(.emit bus "ready" :payload) 5)
  (let [args (await (events/once bus "ready"))]
    (vec args)))     ;; => [:payload]
```

It also rejects if the emitter emits `"error"` first, which makes "start the server, then wait for `listening`" safe.

---

## Async iteration without `for await`

Many Node APIs are **async iterables**: readable streams, `readline` interfaces, `events/on`, directory handles. JavaScript consumes them with `for await`. ClojureScript has no `for await`, but the protocol underneath is simple: get the iterator, then `await` its `.next` until `done`:

```clojure
(defn ^:async iterate-async
  "Calls f on each value of a JS async iterable. Resolves when it's exhausted."
  [^js iterable f]
  (let [it (.call (aget iterable js/Symbol.asyncIterator) iterable)]
    (loop []
      (let [r (await (.next it))]
        (when-not (.-done r)
          (f (.-value r))
          (recur))))))
```

Write this helper once and use it everywhere. Because it awaits each `.next`, the source is consumed one item at a time, and back-pressure works the way the stream expects.

```clojure
(:require ["node:stream" :refer [Readable]])

(await (iterate-async (Readable.from #js [1 2 3]) #(prn :chunk %)))
;; prints: :chunk 1, :chunk 2, :chunk 3
```

`f` is called synchronously. If processing each item is itself async, `await` inside the loop instead of calling `f`, or make the helper `await` its result.

### Reading a file line by line

`readline` over a file stream is the standard way to process a large file without loading it into memory:

```clojure
(:require ["node:fs" :as fs]
          ["node:readline" :as readline])

(defn ^:async each-line [path f]
  (let [rl (readline/createInterface
             #js {:input     (fs/createReadStream path)
                  :crlfDelay js/Infinity})]      ; treat \r\n as one line break
    (await (iterate-async rl f))))

(each-line "lines.txt" #(prn :line %))
;; prints: :line "alpha", :line "beta", :line "gamma"
```

---

## Streams

Node streams move data in chunks: **readable** (a source), **writable** (a sink), and **transform** (both, changing the data on the way through).

### Pipelines

`pipeline` from `node:stream/promises` connects streams, propagates errors from any stage, cleans up all of them on failure, and returns a promise:

```clojure
(:require ["node:stream" :refer [Transform]]
          ["node:stream/promises" :as sp])

(defn ^:async upcase-file [in out]
  (let [upper (Transform.
                #js {:transform (fn [chunk _encoding cb]
                                  (cb nil (.toUpperCase (str chunk))))})]
    (await (sp/pipeline (fs/createReadStream in)
                        upper
                        (fs/createWriteStream out)))))

;; lines.txt holds "alpha\nbeta\ngamma\n"
(await (upcase-file "lines.txt" "upper.txt"))
(fs/readFileSync "upper.txt" "utf8")   ;; => "ALPHA\nBETA\nGAMMA\n"
```

A transform function receives a chunk (a `Buffer` by default) and calls `cb` with an error or `nil`, and the output. A failure in any stage rejects the pipeline promise:

```clojure
(try
  (await (sp/pipeline (fs/createReadStream "missing.txt")
                      (fs/createWriteStream "x.txt")))
  (catch :default e (.-code e)))     ;; => "ENOENT"
```

> ⚠️ **Gotcha:** Don't connect streams with `.pipe`. It doesn't forward errors or destroy the other streams when one fails, so a failure can leak file handles or leave a half-written file with no error reported. Use `pipeline`.

> ⚠️ **Gotcha:** A chunk is an arbitrary slice of the data, not a line or a record. A multi-byte UTF-8 character or a line can be split across two chunks. The upper-casing transform above is safe only because `toUpperCase` on ASCII doesn't care. Use `readline` for lines, or a transform that buffers to a delimiter.

### Object mode

`Readable.from` over a collection of non-strings creates an **object-mode** stream, where each chunk is one value. ClojureScript values pass through unchanged:

```clojure
(await (iterate-async (Readable.from #js [{:a 1}]) #(prn %)))
;; prints: {:a 1}
```

Object-mode streams are a good way to process records with back-pressure: parse, transform and write one item at a time without building the whole result in memory.

---

## Rules of thumb

- Require `EventEmitter` with `:refer`.
- Keep a reference to any listener you'll need to remove.
- Every emitter that can fail gets an `"error"` listener.
- Use `events/once` to turn "wait for this event" into a promise.
- Write `iterate-async` once and use it for every async iterable.
- Connect streams with `pipeline`, never `.pipe`.
- Don't assume chunks line up with lines or characters.
