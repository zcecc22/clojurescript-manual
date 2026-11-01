# 15 — Protocols and records

*After this chapter you can define polymorphic operations with protocols, implement them on records, on existing types (including `nil`, numbers and JS classes), and on one-off objects, and you know when a plain map is still the better choice.*

---

## Protocols

A protocol is a named set of functions dispatched on the type of their first argument. It's the closest thing ClojureScript has to an interface:

```clojure
(defprotocol Shape
  "Things with an area."
  (area [this] "The area.")
  (describe [this] [this prefix]))
```

Each function lists its arities, and the first parameter is always the value being dispatched on. Defining the protocol creates ordinary functions, `area` and `describe`, that you call like any other function.

> **Coming from imperative languages:** Unlike a Java interface, a protocol can be implemented for a type **after the fact**, by code that owns neither the type nor the protocol. Unlike a Go interface, implementation is explicit: you declare it, the compiler doesn't infer it from method names. The method lives in the protocol's namespace, so there's no collision when two protocols both have a `describe`.

---

## Records

`defrecord` defines a type with named fields, optionally implementing protocols:

```clojure
(defrecord Circle [r]
  Shape
  (area [_] (* js/Math.PI r r))
  (describe [this] (describe this "a"))
  (describe [_ prefix] (str prefix " circle of radius " r)))

(defrecord Rect [w h]
  Shape
  (area [_] (* w h))
  (describe [this] (describe this "a"))
  (describe [_ prefix] (str prefix " " w "x" h " rectangle")))
```

Fields are in scope inside the method bodies. Each arity is written as a separate method. Two constructors are generated, positional and from a map:

```clojure
(def c (->Circle 2))
(def r (map->Rect {:w 2 :h 3}))

r                      ;; => #scratch.core.Rect{:w 2, :h 3}
(area r)               ;; => 6
(describe c)           ;; => "a circle of radius 2"
(describe r "the")     ;; => "the 2x3 rectangle"
(map area [r (->Rect 1 1)])   ;; => (6 1)
```

Prefer `->Circle` and `map->Rect` to the raw constructor `(Circle. 2)`.

### Records are maps

A record behaves like a map with its fields as keyword keys:

```clojure
(:r c)                    ;; => 2
(assoc c :r 10)           ;; => #scratch.core.Circle{:r 10}
(assoc c :color :red)     ;; => #scratch.core.Circle{:r 2, :color :red}   extra keys allowed
(map? c)                  ;; => true
(into {} (->Rect 1 2))    ;; => {:w 1, :h 2}
```

So every map function works on records, and code that takes a map takes a record.

> ⚠️ **Gotcha:** `dissoc` of a **declared field** silently turns a record into a plain map, which no longer implements the protocols:
>
> ```clojure
> (dissoc c :r)          ;; => {}
> (type (dissoc c :r))   ;; => cljs.core/PersistentArrayMap
> ```

> ⚠️ **Gotcha:** A record is never `=` to a plain map with the same contents, because the type is part of equality:
>
> ```clojure
> (= c (->Circle 2))     ;; => true
> (= c {:r 2})           ;; => false
> ```

Type tests:

```clojure
(record? c)              ;; => true
(instance? Circle c)     ;; => true
(satisfies? Shape c)     ;; => true
(satisfies? Shape {})    ;; => false
```

Calling a protocol function on a type that doesn't implement it throws:

```clojure
(area {:w 1})
;; Throws: No protocol method Shape.area defined for type cljs.core/PersistentArrayMap: {:w 1}
```

---

## Extending existing types

`extend-protocol` implements one protocol for many types. `extend-type` implements protocols for one type. Neither needs access to the type's source:

```clojure
(extend-protocol Shape
  nil
  (area [_] 0)
  (describe ([_] "nothing") ([_ p] (str p " nothing")))

  number
  (area [n] (* n n))
  (describe ([n] (str "square " n)) ([n p] (str p " square " n))))

(area nil)      ;; => 0
(area 3)        ;; => 9
(describe 3)    ;; => "square 3"
```

Here several arities are written as one method with grouped bodies. JavaScript primitives use lowercase names: `nil`, `number`, `string`, `boolean`, `function`, `object`, `array`. There's also `default`, a fallback for any type that has no implementation:

```clojure
(defprotocol Named (label [x]))

(extend-protocol Named
  string            (label [s] (str "str:" s))
  cljs.core/Keyword (label [k] (str "kw:" (name k)))
  default           (label [x] (str "other:" (pr-str x))))

(label "a")   ;; => "str:a"
(label :b)    ;; => "kw:b"
(label [1])   ;; => "other:[1]"
```

JavaScript classes are extended by their constructor:

```clojure
(extend-type js/Date
  Shape
  (area [_] 0)
  (describe ([d] (str "date " (.getFullYear d))) ([_ p] p)))

(describe (js/Date. 0))   ;; => "date 1970"
```

Extending `nil` is a clean way to give a protocol a safe answer for missing values.

---

## `reify`: one-off implementations

`reify` creates an anonymous object implementing protocols, closing over locals like a function:

```clojure
(def anon
  (reify Shape
    (area [_] 42)
    (describe [_] "reified")
    (describe [_ p] p)))

(area anon)       ;; => 42
```

It's the idiomatic way to build an adapter, a test stub, or an object that some API expects to implement a protocol.

> ⚠️ **Gotcha:** You don't have to implement every method of a protocol. A missing one isn't a compile error, and calling it fails with an unhelpful message:
>
> ```clojure
> (defprotocol Lifecycle (start [this]) (stop [this]))
> (start (reify Lifecycle (stop [_] :stopped)))
> ;; Throws: (intermediate value).scratch$core$Lifecycle$start$arity$1 is not a function
> ```

---

## `deftype`: when you need a bare type

`deftype` defines a type that is **not** a map: no keyword access, no `assoc`, no structural equality unless you implement it. Fields can be mutable, which records' fields can't. It's for building new data structures or low-level objects, not for domain data.

```clojure
(deftype Counter [^:mutable n]
  IDeref
  (-deref [_] n)
  Object
  (inc! [this] (set! n (inc n)) this))

(def ctr (Counter. 0))
(.inc! ctr) (.inc! ctr)
@ctr          ;; => 2      IDeref makes @ work
(map? ctr)    ;; => false
(:n ctr)      ;; => nil
```

`Object` declares plain JavaScript methods, called with `.method`. Implementing `cljs.core` protocols makes a type participate in the core library:

```clojure
(deftype Point [x y]
  IEquiv (-equiv [_ o] (and (instance? Point o) (= x (.-x o)) (= y (.-y o))))
  IHash  (-hash [_] (hash [x y]))
  IPrintWithWriter
  (-pr-writer [_ w _] (-write w (str "#point[" x " " y "]"))))

(Point. 1 2)                               ;; => #point[1 2]
(= (Point. 1 2) (Point. 1 2))              ;; => true
(contains? #{(Point. 1 2)} (Point. 1 2))   ;; => true
```

Useful core protocols include `IDeref`, `ILookup` (keyword access), `ICounted`, `ISeqable`, `IFn` (callable), `IEquiv`/`IHash` (equality, always together) and `IPrintWithWriter`.

---

## Maps, records or types?

| Use | When |
|---|---|
| **A plain map** | The default for domain data. No polymorphism needed |
| **A map plus a multimethod** | Polymorphism by a value in the data, like `:type` (chapter 16) |
| **A record** | Polymorphism by type, with fast protocol dispatch, on map-like data |
| **`reify`** | A one-off implementation that closes over locals |
| **`deftype`** | A new data structure or a mutable low-level object |

Start with maps. Many programs never need a record: data stays plain and functions take maps. Reach for a protocol when you have several types that genuinely need different behaviour for the same operation, and the set of types is open.

---

## Rules of thumb

- Default to plain maps. Introduce a record when protocol dispatch earns it.
- Construct records with `->Name` and `map->Name`.
- Never `dissoc` a record's declared field. `assoc` it to `nil` if you must.
- Extend protocols to `nil` and `default` to centralise fallback behaviour.
- Use `reify` for adapters and stubs, and implement every method.
- Reserve `deftype` for data structures. If it's domain data, it's a map or a record.
