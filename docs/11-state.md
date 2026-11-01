# 11 — State

*After this chapter you can manage mutable state with atoms, react to changes with watches, enforce invariants with validators, and keep state across hot reloads.*

---

## Values don't change, references do

Everything so far has been immutable. Real programs still have state: a cache, a connection pool, a counter, the set of open sessions. ClojureScript's answer keeps the data immutable and makes the **reference** to it mutable. An **atom** holds one value at a time, and you change which value it holds:

```clojure
(def counter (atom 0))

@counter               ;; => 0     @ is shorthand for (deref counter)
(swap! counter inc)    ;; => 1     apply a function to the current value
(swap! counter + 10)   ;; => 11    extra args are passed to the function
(reset! counter 100)   ;; => 100   replace the value outright
```

The value inside is still an ordinary immutable value. Whoever dereferenced the atom earlier holds the old value, and it never changes under them.

> **Coming from imperative languages:** An atom is the one variable you're allowed to reassign. Where you'd have a mutable field or a package-level `var`, use an atom, and only there. The `!` at the end of `swap!` and `reset!` is a naming convention for "this function mutates something".

---

## One atom, structured data

The idiomatic shape is one atom holding a map, not many atoms:

```clojure
(def state (atom {:users {} :next-id 1}))

(defn add-user! [name]
  (swap! state
         (fn [{:keys [next-id] :as s}]
           (-> s
               (assoc-in [:users next-id] {:name name})
               (update :next-id inc)))))

(add-user! "Ada")
(add-user! "Bob")
@state   ;; => {:users {1 {:name "Ada"}, 2 {:name "Bob"}}, :next-id 3}
```

The function passed to `swap!` takes the old state and returns the new one. Several related fields change together, in a single step, so no code ever sees a half-updated state. The function itself is pure and can be tested without the atom.

### `swap!` variants

```clojure
(swap-vals! counter inc)        ;; => [100 101]   [old new]
(reset-vals! counter 0)         ;; => [101 0]
(compare-and-set! counter 0 5)  ;; => true        set only if currently 0
(compare-and-set! counter 0 9)  ;; => false
```

`swap-vals!` is the one to reach for when you need to know what the state was. For example, "pop the first job from the queue" needs the old value to know which job you took.

> **Coming from imperative languages:** In JVM Clojure, `swap!` can retry its function under contention, which is why the function must be free of side effects. JavaScript is single-threaded, so in ClojureScript the function runs exactly once per `swap!`. Keep it pure anyway: code that's correct on one thread is easier to reason about and to move.

---

## Watches

A watch is a function called after every change:

```clojure
(add-watch counter :log
  (fn [key ref old new]
    (println "watch" key old "->" new)))

(swap! counter inc)    ;; prints: watch :log 6 -> 7
(reset! counter 6)     ;; prints: watch :log 7 -> 6

(remove-watch counter :log)
```

The key identifies the watch so you can remove or replace it. Watches suit derived side effects, such as persisting state, logging changes or invalidating a cache, because the code that changes the state doesn't need to know about them.

Watches fire on every `swap!` and `reset!`, even when the new value equals the old. Compare `old` and `new` if that matters.

---

## Validators

A validator is a predicate that every new value must pass. A failing update throws and leaves the atom unchanged:

```clojure
(def positive (atom 1 :validator pos?))

(reset! positive -1)   ;; Throws: Validator rejected reference state
@positive              ;; => 1

(set-validator! counter number?)   ; add one to an existing atom
```

Validators are a cheap way to enforce an invariant on central state, and spec (chapter 19) makes them expressive: `:validator #(s/valid? ::state %)`.

---

## Volatiles

A volatile is a stripped-down atom with no watches and no validators. It's faster, and meant for local mutable state inside a function, such as the internals of a transducer (chapter 18):

```clojure
(def v (volatile! 0))
(vswap! v inc)
(vreset! v 10)
@v                     ;; => 10
```

Don't use volatiles for application state. Use them only where an atom's features are unnecessary and its cost shows up in a profile.

---

## `delay`: compute once, on demand

`delay` defers a computation until it's first dereferenced, then caches the result:

```clojure
(def d (delay (println "computing...") 42))

(realized? d)   ;; => false
@d              ;; prints: computing...   => 42
@d              ;; => 42, nothing printed
```

This is the lazy-initialisation pattern: configuration loaded on first use, an expensive client created only if something needs it.

---

## `defonce` and hot reload

When shadow-cljs reloads a file, every `def` runs again. That would replace your state atom with a fresh one. `defonce` only defines the var if it isn't defined already:

```clojure
(defonce conn (atom {:opened-at :first-load}))
(defonce conn (atom {:opened-at :second-load}))   ; ignored: conn exists

@conn   ;; => {:opened-at :first-load}
```

Use `defonce` for state atoms and for anything that holds a resource, like a server or a database connection, so reloading code doesn't lose state or open a second server.

---

## Dynamic vars

A var marked `^:dynamic` can be temporarily rebound with `binding`:

```clojure
(def ^:dynamic *level* :info)
(defn current [] *level*)

(current)                            ;; => :info
(binding [*level* :debug] (current)) ;; => :debug
(current)                            ;; => :info
```

The earmuffs (`*level*`) are the naming convention. Dynamic vars are for ambient configuration, like `*print-length*`, and are rare in application code.

> ⚠️ **Gotcha:** `binding` only lasts while its body runs **synchronously**. Anything that runs later sees the original value. That includes lazy sequences realised after the `binding` returns:
>
> ```clojure
> (binding [*level* :debug] (map (fn [_] (current)) [1 2]))
> ;; => (:info :info)
> ```
>
> and promise callbacks and timers:
>
> ```clojure
> (binding [*level* :debug]
>   (.then (js/Promise.resolve 1) (fn [_] (prn (current)))))
> ;; prints: :info
> ```
>
> For anything asynchronous, pass configuration as an argument.

---

## Where state should live

The goal is to keep state small and at the edges:

- **Pure functions in the middle.** They take the state as a value and return a new value. All of your logic lives here and is trivial to test.
- **A few atoms at the boundary.** Each owns one area: the app state, a cache, a connection.
- **`swap!` calls thin.** `(swap! state add-user "Ada")` with a pure `add-user`, not a hundred-line anonymous function.

> **Coming from imperative languages:** There are no refs, agents or STM in ClojureScript, and no locks. On a single thread, the atom is the only reference type you need. Concurrency in Node comes from asynchronous I/O, not threads (chapter 14).

---

## Rules of thumb

- One atom per area of state, holding a map. Not one atom per field.
- Pass a pure function to `swap!`, and name it so it can be tested.
- Use `swap-vals!` when you need the old value.
- Use watches for derived side effects and validators for invariants.
- `defonce` every atom and resource that should survive a reload.
- Don't rely on `binding` across lazy sequences, promises or callbacks.
