# 14 — Async patterns

*After this chapter you can reason about the event loop, add timeouts and cancellation to any async operation, bound concurrency, retry with backoff, move CPU-heavy work off the main thread, and shut down cleanly.*

---

## One thread, one event loop

Your ClojureScript code runs on a single thread. Concurrency comes from **not waiting**: an I/O call hands the work to Node, your code continues, and the callback (or `await` continuation) runs later on the same thread.

The consequences:

- **No data races.** Between two `await`s, nothing else runs. A `swap!` is never interleaved with another.
- **No parallel CPU work.** Two CPU-bound computations take twice as long, whatever you do with promises.
- **Blocking blocks everything.** A long synchronous loop delays every timer, request and callback in the process.

```clojure
(let [t0 (js/Date.now)]
  (js/setTimeout #(prn :timer-late-by (- (js/Date.now) t0 10)) 10)
  (loop [] (when (< (- (js/Date.now) t0) 100) (recur))))   ; 100 ms of busy work
;; prints: :timer-late-by 90
```

The 10 ms timer fired 90 ms late because the loop held the thread.

> **Coming from imperative languages:** Go's goroutines and Java's threads run on many cores, and you protect shared state with locks. Here the trade is reversed: shared state is safe by default, and you protect *latency* by keeping each synchronous stretch short. If you need parallel CPU, see worker threads below.

### Ordering is a detail. Don't depend on it

Callbacks queued in the same tick run in an order that depends on where you are:

```clojure
(defn order []
  (println "sync start")
  (js/setTimeout #(println "setTimeout 0") 0)
  (js/setImmediate #(println "setImmediate"))
  (.then (js/Promise.resolve) #(println "promise microtask"))
  (js/process.nextTick #(println "nextTick"))
  (println "sync end"))
```

Called at the top level of the script, this printed `nextTick` before `promise microtask`. Called from inside an async function, after an `await`, the same function printed `promise microtask` first. And `setTimeout 0` vs `setImmediate` swapped between runs (once in five). The only guarantee is that synchronous code finishes first. If one callback must run after another, chain them.

---

## Timeouts

Nothing in a promise has a timeout built in. `Promise.race` against a rejecting timer adds one:

```clojure
(defn with-timeout [p ms]
  (js/Promise.race
    #js [p
         (js/Promise. (fn [_ reject]
                        (js/setTimeout #(reject (ex-info "timeout" {:type :timeout :ms ms}))
                                       ms)))]))

(try (await (with-timeout (tp/setTimeout 100 :slow) 10))
     (catch :default e (ex-data e)))     ;; => {:type :timeout, :ms 10}

(await (with-timeout (tp/setTimeout 1 :fast) 50))   ;; => :fast
```

`tp` here is `node:timers/promises`, whose `setTimeout` returns a promise that resolves to its second argument after the delay.

> ⚠️ **Gotcha:** `with-timeout` stops *waiting*. It doesn't stop the *work*. The slow operation keeps running, holding its socket or file handle, and its result is thrown away. To actually cancel, the operation has to accept an `AbortSignal`.

---

## Cancellation with `AbortController`

`AbortController` is the standard cancellation mechanism. You pass its `signal` to an operation, and calling `.abort` cancels it. Node's own APIs accept signals: `fetch`, `fs.readFile`, `timers/promises`, `events.once`, `child_process`, stream `pipeline`.

```clojure
(:require ["node:timers/promises" :as tp])

(let [ac (js/AbortController.)]
  (js/setTimeout #(.abort ac) 5)
  (try
    (await (tp/setTimeout 1000 :done #js {:signal (.-signal ac)}))
    (catch :default e [(.-name e) (.-code e)])))
;; => ["AbortError" "ABORT_ERR"]
```

For a timeout that genuinely cancels, use `AbortSignal.timeout`:

```clojure
(await (tp/setTimeout 1000 nil #js {:signal (js/AbortSignal.timeout 10)}))
;; Throws: an error whose .-name is "AbortError"
```

Thread a signal through your own async functions as an option, and check `(.-aborted signal)` between steps of long-running work. That's the cooperative cancellation Go programmers know from `context.Context`.

---

## Bounded concurrency

`Promise.all` over a thousand items starts a thousand operations at once, which exhausts file handles, overwhelms a database, or gets you rate-limited. A small worker pool caps it. N workers each pull the next index until none are left:

```clojure
(defn ^:async limited-map
  "Like (map f xs) for an async f, with at most `limit` calls in flight.
  Resolves to a vector of results in input order."
  [limit f xs]
  (let [items   (vec xs)
        results (make-array (count items))
        next-i  (atom 0)
        worker  (fn ^:async worker []
                  (loop []
                    (let [i (dec (swap! next-i inc))]
                      (when (< i (count items))
                        (aset results i (await (f (nth items i))))
                        (recur)))))]
    (await (js/Promise.all (into-array (repeatedly (min limit (count items)) worker))))
    (vec results)))
```

With a job that tracks how many calls are in flight:

```clojure
(await (limited-map 3 job (range 10)))
;; => [0 1 4 9 16 25 36 49 64 81]     peak in-flight: 3
```

The shared `atom` is safe without any locking because only one worker runs at a time between `await`s. The same pool in Go would need a mutex or a channel.

If any call rejects, `Promise.all` rejects, but the other workers keep running their current items. Use `allSettled` inside the worker if you need every result.

---

## Retry with backoff

```clojure
(defn ^:async retry
  "Calls (f) until it resolves, at most `attempts` times, doubling the delay each time.
  An error for which (retry? e) is false is rethrown immediately."
  [f {:keys [attempts base-ms retry?] :or {attempts 3 base-ms 10 retry? (constantly true)}}]
  (loop [n 1]
    (let [r (try
              {:ok (await (f))}
              (catch :default e
                (if (and (< n attempts) (retry? e)) {:retry e} (throw e))))]
      (if (contains? r :ok)
        (:ok r)
        (do (await (tp/setTimeout (* base-ms (js/Math.pow 2 (dec n)))))
            (recur (inc n)))))))
```

A flaky operation that fails twice and then succeeds:

```clojure
(await (retry flaky {}))   ;; retries after 10 ms, then 20 ms  => :ok
```

`recur` can't appear inside `try` or `catch` (the compiler says `Can't recur here`), so the `try` returns a map saying what to do and the decision happens outside it. That's a pattern worth remembering whenever a loop needs to recover from errors.

Only retry operations that are safe to repeat, and only for errors that might be transient: a timeout or a 503, not a 400. Pass a `:retry?` predicate to say which errors those are (chapter 25 has one for HTTP). In production, add jitter to the delay so many clients don't retry in lockstep.

---

## Worker threads for CPU work

`node:worker_threads` runs JavaScript on another thread with its own event loop and memory. Workers communicate by message passing:

```clojure
(:require ["node:worker_threads" :as wt]
          ["node:events" :as events])

(let [w (wt/Worker.
          "const {parentPort, workerData} = require('node:worker_threads');
           let s = 0; for (let i = 0; i < workerData; i++) s += i;
           parentPort.postMessage(s);"
          #js {:eval true :workerData 1e7})]
  (aget (await (events/once w "message")) 0))
;; => 49999995000000
```

A real worker runs a script file. With shadow-cljs, that's a second `:node-script` build whose `:main` reads `wt/workerData` and posts to `wt/parentPort` (chapter 21 covers multiple builds). Messages are copied with the structured clone algorithm, so send plain JS data (`clj->js`) and convert back on the other side.

Reach for workers only for real CPU-bound work: hashing, compression, parsing large files, image processing. For I/O they add cost and no benefit.

---

## Graceful shutdown

A server should stop accepting work, finish what's in flight, release resources, and exit on `SIGTERM` (from a process manager or container runtime) and `SIGINT` (Ctrl-C):

```clojure
(defn shutdown! [server]
  (prn :shutting-down)
  (.close server (fn [] (prn :closed)))    ; then release anything else you hold
  ;; force exit if connections don't drain in time
  (.unref (js/setTimeout #(js/process.exit 1) 10000)))

(defn install-signal-handlers! [server]
  (doseq [sig ["SIGINT" "SIGTERM"]]
    (.once js/process sig #(shutdown! server))))
```

There's no `process.exit` on the success path. Once the server is closed and nothing else is open, Node exits by itself with code 0, after flushing output, which `process.exit` doesn't wait for (chapter 26). `.unref` on the deadline timer means it won't keep the process alive by itself. Node exits when nothing is left on the event loop: no timers, sockets or pending I/O. A process that won't exit is holding something open. Chapter 24 applies this to an HTTP server.

---

## Rules of thumb

- Keep synchronous stretches short. Any loop over large data either chunks its work across ticks or moves to a worker.
- Never depend on callback ordering within a tick. Chain what must happen in order.
- Every network call gets a timeout. Prefer `AbortSignal.timeout`, which actually cancels.
- Accept an `AbortSignal` in your own long-running async functions.
- Bound concurrency with a worker pool. A bare `Promise.all` over unbounded input is a bug.
- Retry only idempotent operations, and only transient errors, with exponential backoff.
- Handle `SIGINT` and `SIGTERM`: stop accepting work, drain, then exit.
