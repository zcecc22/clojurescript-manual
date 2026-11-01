# 06 — Collections

*After this chapter you can model data with persistent vectors, maps and sets, update nested structures without mutation, and pick the right collection for each job.*

---

## Persistent means immutable, and cheap

The four core collections are **immutable**. Operations that "change" one return a new collection and leave the original untouched:

```clojure
(def v [1 2 3])
(conj v 4)   ;; => [1 2 3 4]
v            ;; => [1 2 3]
```

The new collection isn't a full copy. It shares most of its internal structure with the old one, so updating one element of a million-element vector creates a handful of small nodes, not a million-element copy:

```clojure
(let [big  (vec (range 1000000))
      big2 (assoc big 0 :x)]
  [(first big) (first big2)])   ;; => [0 :x]   both versions exist, sharing almost everything
```

The practical consequence is that you can hand a collection to any function, store it anywhere, and use it as a map key, and nothing will ever change it behind your back. No defensive copies, no "who else holds a reference to this?"

> **Coming from imperative languages:** In Go, appending to a slice may or may not alias the original. In Python and JavaScript, passing a list or object lets the callee mutate it. None of that applies here. The cost is that updating a collection means using the return value: `(conj v 4)` on its own line does nothing useful.

---

## Vectors

The everyday ordered collection. Indexed access and adding to the end are fast:

```clojure
(def v [1 2 3])

(conj v 4)         ;; => [1 2 3 4]   conj adds at the END of a vector
(nth v 0)          ;; => 1
(get v 10)         ;; => nil
(v 1)              ;; => 2           vectors are functions of their index
(assoc v 1 :x)     ;; => [1 :x 3]
(assoc v 3 :end)   ;; => [1 2 3 :end]   one past the end appends
(peek v)           ;; => 3           last element
(pop v)            ;; => [1 2]
(subvec v 1)       ;; => [2 3]
```

`nth` with an out-of-range index throws unless you give it a default. `get` returns `nil`:

```clojure
(nth v 10)         ;; Throws: No item 10 in vector of length 3
(nth v 10 :nf)     ;; => :nf
(assoc v 5 :x)     ;; Throws: Index 5 out of bounds  [0,3]
```

## Lists

Lists are linked lists, and `conj` adds at the **front**:

```clojure
(conj '(1 2) 0)    ;; => (0 1 2)
(conj [1 2] 0)     ;; => [1 2 0]
```

You'll rarely build lists yourself. They're mostly code (every `(f x)` is a list) and the printed form of sequences. Use vectors for data.

> ⚠️ **Gotcha:** `conj` adds wherever it's efficient for the collection: the end of a vector, the front of a list. Code that gets a sequence back from `map` or `filter` and then `conj`es onto it is adding to the front. If order matters, convert with `vec` or `(into [] ...)` first.

---

## Maps

Maps are the workhorse. Most domain data is a map with keyword keys:

```clojure
(def m {:name "Ada" :age 36})

(assoc m :lang "en")    ;; => {:name "Ada", :age 36, :lang "en"}
(dissoc m :age)         ;; => {:name "Ada"}
(update m :age inc)     ;; => {:name "Ada", :age 37}
(update m :age + 10)    ;; => {:name "Ada", :age 46}   extra args go to the function
(get m :x :default)     ;; => :default
(contains? m :age)      ;; => true
(find m :age)           ;; => [:age 36]
```

Combining and slicing:

```clojure
(merge {:a 1 :b 2} {:b 3 :c 4})        ;; => {:a 1, :b 3, :c 4}   rightmost wins
(merge-with + {:a 1 :b 2} {:b 3})      ;; => {:a 1, :b 5}
(select-keys m [:name :zzz])           ;; => {:name "Ada"}
(zipmap [:a :b] [1 2])                 ;; => {:a 1, :b 2}
(into {} [[:a 1] [:b 2]])              ;; => {:a 1, :b 2}
(keys m)                               ;; => (:name :age)
(vals m)                               ;; => ("Ada" 36)
```

Seen as a sequence, a map is a sequence of `[key value]` entries:

```clojure
(first {:a 1 :b 2})      ;; => [:a 1]
(into [] {:a 1})         ;; => [[:a 1]]
```

> ⚠️ **Gotcha:** Maps are unordered. Small maps (up to eight entries) happen to keep insertion order, because they're stored as an array internally (`PersistentArrayMap`). Add a ninth key and the map becomes a `PersistentHashMap`, and the order is gone. Code that relies on map order works in tests with small maps and breaks in production. Use `sorted-map` if you need ordering, or a vector of pairs.

### `fnil` for missing keys

`update` on a missing key passes `nil` to your function. `fnil` wraps a function to substitute a default for `nil`:

```clojure
(update {} :count (fnil inc 0))          ;; => {:count 1}
(update {:count 1} :count (fnil inc 0))  ;; => {:count 2}
```

> ⚠️ **Gotcha:** `(update {} :xs conj 1)` returns `{:xs (1)}`, a **list**, because `(conj nil 1)` makes a list. If you want a vector, start it explicitly: `(update {} :xs (fnil conj []) 1)`.

---

## Nested data

Real data nests. The `-in` functions take a path, a vector of keys:

```clojure
(def db {:users {1 {:name "Ada" :tags #{:admin}}}})

(get-in db [:users 1 :name])                 ;; => "Ada"
(assoc-in db [:users 2 :name] "Bob")         ;; creates intermediate maps as needed
;; => {:users {1 {:name "Ada", :tags #{:admin}}, 2 {:name "Bob"}}}
(update-in db [:users 1 :tags] conj :ops)
;; => {:users {1 {:name "Ada", :tags #{:admin :ops}}}}
```

Paths can include vector indices too: `(get-in {:xs [10 20]} [:xs 1])` is `20`.

All of these accept `nil` and treat it as an empty map, so they compose well with missing data:

```clojure
(assoc nil :a 1)              ;; => {:a 1}
(get-in nil [:a :b])          ;; => nil
(update nil :n (fnil inc 0))  ;; => {:n 1}
```

---

## Sets

```clojure
(def s #{:a :b})

(conj s :c)           ;; => #{:b :a :c}
(disj s :a)           ;; => #{:b}
(contains? s :a)      ;; => true
(s :a)                ;; => :a       sets are functions: element or nil
(s :z)                ;; => nil
(set [1 1 2])         ;; => #{1 2}
```

Set algebra lives in `clojure.set`:

```clojure
(set/union s #{:z})            ;; => #{:b :a :z}
(set/intersection s #{:a :q})  ;; => #{:a}
(set/difference s #{:a})       ;; => #{:b}
```

Because a set is a function returning the element or `nil`, it's a ready-made membership predicate: `(filter #{:admin :ops} roles)`.

> ⚠️ **Gotcha:** `contains?` checks for a **key**, and a vector's keys are its indices:
>
> ```clojure
> (contains? [10 20 30] 1)    ;; => true    there is an index 1
> (contains? [10 20 30] 20)   ;; => false
> (some #{20} [10 20 30])     ;; => 20      what you meant
> ```

---

## Sorted collections and queues

```clojure
(sorted-map :c 3 :a 1 :b 2)    ;; => {:a 1, :b 2, :c 3}
(sorted-set 3 1 2)             ;; => #{1 2 3}
(sorted-map-by > 1 :a 3 :c)    ;; => {3 :c, 1 :a}

(def q (conj #queue [] 1 2 3)) ;; FIFO: conj at the back, peek/pop at the front
(peek q)                       ;; => 1
(pop q)                        ;; => #queue [2 3]
```

---

## Equality and hashing

Collections compare by value, deeply (chapter 02), and their hashes are consistent with that. Any value, including a collection, can be a map key or a set element:

```clojure
({[1 2] :vec-key} [1 2])          ;; => :vec-key
(get {{:a 1} :map-key} {:a 1})    ;; => :map-key
```

Composite keys like `[user-id date]` are idiomatic and need no special handling.

---

## Choosing a collection

| You need | Use |
|---|---|
| Ordered items, indexed access, append | vector `[]` |
| Named fields, lookup by key | map `{}` with keyword keys |
| Membership, uniqueness | set `#{}` |
| Ordering by key | `sorted-map`, `sorted-set` |
| FIFO | `#queue []` |
| Code, or prepend-only stacks | list `'()` (rarely) |

Predicates for checking which you have: `map?`, `vector?`, `set?`, `list?`, `coll?`, `sequential?` (vectors and lists, not maps), `associative?` (maps and vectors).

---

## Rules of thumb

- Model entities as maps with keyword keys. Don't reach for records or types until you need polymorphism (chapter 15).
- Always use the return value. `(assoc m :k v)` doesn't change `m`.
- Use vectors for ordered data. Convert sequences back with `vec` or `into []` when order matters.
- Never rely on map ordering, even if your test maps keep it.
- Use `get-in`, `assoc-in` and `update-in` for nested data, and `fnil` when the key may be missing.
- `contains?` takes a key. To search a vector for a value, use `some` with a set.
