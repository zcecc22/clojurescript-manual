# 04 — Functions

*After this chapter you can write every function shape ClojureScript supports, destructure arguments, and compose functions instead of writing glue code.*

---

## Defining functions

```clojure
(defn greet
  "Returns a greeting."
  ([] (greet "world"))
  ([name] (str "Hello, " name)))

(greet)        ;; => "Hello, world"
(greet "Ada")  ;; => "Hello, Ada"
```

`defn` takes a name, an optional docstring, and either one parameter vector with a body or several arities, each wrapped in its own list. One arity calling another is the idiomatic way to supply defaults, because there are no default parameter values.

The single-arity form is what you'll write most:

```clojure
(defn area [w h]
  (* w h))
```

The body's last expression is the return value. There's no `return` keyword.

Calling with the wrong number of arguments is a compiler warning where the compiler can see it, and an error at runtime:

```clojure
(greet 1 2)   ;; Throws: Invalid arity: 2
```

> **Coming from imperative languages:** Functions are values, the same as numbers. `defn` is shorthand for `(def greet (fn [...] ...))`: it creates an anonymous function and binds a name to it. Passing functions to functions is routine here, not an advanced technique.

---

## Variadic functions

`&` collects the remaining arguments into a sequence:

```clojure
(defn sum [& xs]
  (reduce + 0 xs))

(sum)          ;; => 0
(sum 1 2 3)    ;; => 6
```

With no rest arguments the parameter is `nil`, not an empty list:

```clojure
(defn log [level & msgs] [level msgs])

(log :info)          ;; => [:info nil]
(log :info "a" "b")  ;; => [:info ("a" "b")]
```

`apply` goes the other way: it spreads a collection into arguments. Arguments before the collection are passed as-is:

```clojure
(apply sum [1 2 3])     ;; => 6
(apply + 1 2 [3 4])     ;; => 10
```

---

## Anonymous functions

`fn` creates a function without a name:

```clojure
(def double-it (fn [x] (* 2 x)))
(double-it 4)   ;; => 8
```

For short one-off functions, `#()` is a reader shorthand. `%` is the single argument, `%1`, `%2` and so on are positional, and `%&` is the rest:

```clojure
(map #(* % %) [1 2 3])           ;; => (1 4 9)
(map #(+ %1 %2) [1 2] [10 20])   ;; => (11 22)
```

Keep `#()` to one short expression. It can't be nested, and once it needs a name or a second line, `fn` reads better.

> ⚠️ **Gotcha:** `#([% %])` does not return a vector. The body of `#()` is always a call, so it tries to call the vector `[% %]` as a function. Write `#(vector % %)` or `(fn [x] [x x])`.

---

## Destructuring

Destructuring pulls values out of collections right in the parameter list or in `let`. It works the same in both places.

### Sequential destructuring

```clojure
(let [[a b & more] [1 2 3 4]]
  [a b more])            ;; => [1 2 (3 4)]

(let [[a _ c] [1 2 3]]
  [a c])                 ;; => [1 3]     _ by convention means "ignored"

(let [[x y] [1]]
  [x y])                 ;; => [1 nil]   missing positions are nil, not an error
```

### Map destructuring

`:keys` binds local names to the keyword keys of the same name. `:or` gives defaults and `:as` keeps the whole map:

```clojure
(let [{:keys [name age] :or {age 0} :as user} {:name "Ada"}]
  [name age user])
;; => ["Ada" 0 {:name "Ada"}]
```

Other shapes:

```clojure
(let [{n :name} {:name "Ada"}] n)                      ;; => "Ada"     rename
(let [{:strs [a]} {"a" 1}] a)                          ;; => 1         string keys
(let [{:keys [user/id]} {:user/id 7}] id)              ;; => 7         namespaced keys
(let [{{:keys [city]} :address} {:address {:city "Paris"}}]
  city)                                                ;; => "Paris"   nested
```

In a parameter list, map destructuring gives you named arguments:

```clojure
(defn area [{:keys [w h]}]
  (* w h))

(area {:w 2 :h 3})   ;; => 6
```

This is the idiomatic way to pass more than three or four arguments. Callers name each value, the order doesn't matter, and adding a field doesn't break existing callers.

> ⚠️ **Gotcha:** `:or` defaults only apply to the destructured locals. The `:as` binding is the original map, without the defaults.

### Keyword arguments

Destructuring the rest arguments as a map gives keyword arguments:

```clojure
(defn connect [host & {:keys [port] :or {port 80}}]
  [host port])

(connect "h")              ;; => ["h" 80]
(connect "h" :port 8080)   ;; => ["h" 8080]
(connect "h" {:port 9})    ;; => ["h" 9]     a trailing map also works (CLJS 1.11+)
```

Most code passes a single options map instead, which is easier to build up and pass along.

---

## Closures

Functions capture the locals in scope when they're created:

```clojure
(defn make-counter []
  (let [n (atom 0)]
    (fn [] (swap! n inc))))

(let [c (make-counter)]
  (c) (c) (c))   ;; => 3
```

The locals themselves are immutable. To have state that changes, the closure captures an atom (chapter 11).

> **Coming from imperative languages:** The classic loop-closure bug, where every callback sees the last value of `i`, can't happen here. Bindings never change, so each closure captures the value it saw.

---

## Higher-order functions

`cljs.core` has a set of functions that build functions. Learn these and a lot of tiny `fn`s disappear:

```clojure
((partial + 10) 5)                ;; => 15          fix leading arguments
((comp str inc) 1)                ;; => "2"         right to left: inc, then str
((juxt :a :b) {:a 1 :b 2})        ;; => [1 2]       several functions, one vector
((complement even?) 3)            ;; => true        negate a predicate
((constantly 7) :anything)        ;; => 7
(identity 3)                      ;; => 3
```

`memoize` caches results by argument:

```clojure
(def slow-sq (memoize (fn [x] (println "computing" x) (* x x))))
(slow-sq 3)   ;; prints: computing 3
(slow-sq 3)   ;; => 9, nothing printed
```

The cache is unbounded and lives forever, so only memoize functions with a small input domain.

Keywords, maps, sets and vectors can all be called as functions. `fn?` is true only for real functions, while `ifn?` is true for anything callable:

```clojure
(fn? :a)    ;; => false
(ifn? :a)   ;; => true
```

---

## Threading macros

Nested calls read inside out. Threading macros let you write them as a pipeline:

```clojure
;; Without threading
(map inc (filter even? [1 2 3 4]))

;; ->> threads the value in as the LAST argument
(->> [1 2 3 4]
     (filter even?)
     (map inc))            ;; => (3 5)

;; -> threads the value in as the FIRST argument
(-> {:a 1}
    (assoc :b 2)
    (dissoc :a))           ;; => {:b 2}
```

The rule for picking one: functions on *collections* (`assoc`, `update`, `get`) take the collection first, so use `->`. Functions on *sequences* (`map`, `filter`, `reduce`) take it last, so use `->>`.

Variations:

```clojure
(some-> {:a {:b 1}} :a :b inc)   ;; => 2     stops at the first nil
(some-> {:a nil} :a :b inc)      ;; => nil   instead of calling (inc nil)
(cond-> 1 true inc false (* 100)) ;; => 2    applies only the steps whose test is truthy
(as-> 5 x (* x 2) (- 100 x))      ;; => 90   names the value, for when its position varies
```

---

## Recursion has no tail-call optimisation

JavaScript engines don't eliminate tail calls, so a recursive function uses one stack frame per call. On Node 20 with the default stack, a trivial self-recursive function overflowed somewhere between 7,000 and 9,000 levels:

```clojure
(defn countdown [n]
  (if (zero? n) :done (countdown (dec n))))

(countdown 9000)   ;; Throws: Maximum call stack size exceeded
```

`recur` fixes this. It's a jump back to the top of the function (or of a `loop`) and uses no stack:

```clojure
(defn countdown [n]
  (if (zero? n) :done (recur (dec n))))

(countdown 1000000)   ;; => :done
```

The compiler checks that `recur` is in tail position, and refuses to compile if it isn't. Chapter 05 covers `loop`/`recur` properly.

Pre- and post-conditions are available as a map right after the parameter vector. They throw an `Assert failed` error, and they're removed when `*assert*` is false, which is how release builds usually run:

```clojure
(defn pos-sqrt [x]
  {:pre [(pos? x)]}
  (js/Math.sqrt x))

(pos-sqrt -1)   ;; Throws: Assert failed: (pos? x)
```

---

## Rules of thumb

- Use multiple arities to provide defaults.
- More than three or four parameters? Take a map and destructure it.
- Use `#()` only for one short expression. Anything else gets `fn`.
- Use `->` for collection operations and `->>` for sequence pipelines.
- Self-recursion uses `recur` unless the depth is guaranteed to be small.
- Reach for `partial`, `comp` and `juxt` before writing a wrapper `fn`.
