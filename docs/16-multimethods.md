# 16 — Multimethods

*After this chapter you can dispatch on any property of your data, not just its type, build open hierarchies of keywords, and choose between multimethods and protocols.*

---

## Dispatch on a value

A multimethod has a **dispatch function** that computes a value from the arguments, and **methods** registered for particular dispatch values:

```clojure
(defmulti handle-event :type)

(defmethod handle-event :user/signup [{:keys [email]}]
  (str "welcome " email))

(defmethod handle-event :user/delete [{:keys [id]}]
  (str "deleted " id))

(defmethod handle-event :default [e]
  (str "ignored " (:type e)))

(handle-event {:type :user/signup :email "a@b.c"})   ;; => "welcome a@b.c"
(handle-event {:type :user/delete :id 7})            ;; => "deleted 7"
(handle-event {:type :other})                        ;; => "ignored :other"
```

Here the dispatch function is the keyword `:type`, so the data itself says how it should be handled. This is the natural polymorphism for plain maps: events, messages, commands, AST nodes, anything with a tag.

Without a `:default` method, an unknown dispatch value throws:

```clojure
(strict-handle {:type :b})
;; Throws: No method in multimethod 'scratch.core/strict-handle' for dispatch value: :b
```

Methods can be added from any namespace, so the set of handled cases is open: a plugin namespace can add `(defmethod handle-event :billing/charged ...)` without touching the original.

> **Coming from imperative languages:** This replaces the `switch` on a type field that grows with every new case. Each case becomes a separate `defmethod` that can live next to the code it concerns, and adding a case doesn't modify existing code.

---

## Dispatch on anything

The dispatch function can be any function of all the arguments. Returning a vector dispatches on several things at once:

```clojure
(defmulti fee
  (fn [account amount]
    [(:tier account) (if (> amount 1000) :large :small)]))

(defmethod fee [:gold :large] [_ _] 0)
(defmethod fee [:gold :small] [_ _] 1)
(defmethod fee :default [_ amt] (* 0.01 amt))

(fee {:tier :gold} 5000)    ;; => 0
(fee {:tier :gold} 10)      ;; => 1
(fee {:tier :basic} 500)    ;; => 5
```

Dispatching on `type` gives class-based dispatch, though a protocol is usually a better fit for that:

```clojure
(defmulti by-class type)
(defmethod by-class js/Number [_] :number)
(defmethod by-class js/String [_] :string)
(defmethod by-class PersistentVector [_] :vector)

(by-class 1)     ;; => :number
(by-class [1])   ;; => :vector
```

---

## Hierarchies

`derive` builds an "is-a" relationship between namespaced keywords, with no classes involved:

```clojure
(derive ::cat ::animal)
(derive ::dog ::animal)
(derive ::lion ::cat)

(isa? ::lion ::animal)   ;; => true
(isa? ::dog ::cat)       ;; => false
(ancestors ::lion)       ;; => #{:scratch.core/cat :scratch.core/animal}
```

Multimethods dispatch through the hierarchy, picking the most specific method:

```clojure
(defmulti speak :kind)
(defmethod speak ::animal [_] "...")
(defmethod speak ::cat [_] "meow")

(speak {:kind ::lion})   ;; => "meow"   lion isa cat
(speak {:kind ::dog})    ;; => "..."    dog isa animal
```

A keyword can derive from several parents. When two methods match equally, the call throws until you state a preference:

```clojure
(derive ::robocat ::cat)
(derive ::robocat ::machine)
(defmethod speak ::machine [_] "beep")

(speak {:kind ::robocat})
;; Throws: Multiple methods in multimethod 'scratch.core/speak' match dispatch value:
;;   :scratch.core/robocat -> :scratch.core/machine and :scratch.core/cat, and neither is preferred

(prefer-method speak ::cat ::machine)
(speak {:kind ::robocat})   ;; => "meow"
```

The hierarchy is global by default. `derive` mutates it, so do your `derive` calls at namespace load time, not at runtime. Hierarchies are useful but optional. Most multimethods dispatch on plain keywords with no `derive` at all.

---

## Introspection

```clojure
(keys (methods handle-event))              ;; => (:user/signup :user/delete :default)
(get-method handle-event :user/signup)     ;; the function, or nil
(remove-method handle-event :user/delete)
(handle-event {:type :user/delete :id 7})  ;; => "ignored :user/delete"
```

> ⚠️ **Gotcha:** `defmulti` behaves like `defonce`. Evaluating it again, as hot reload or your REPL does, **does not replace the dispatch function**:
>
> ```clojure
> (defmulti area-of (fn [shape] (:shape shape)))
> (defmethod area-of :circle [{:keys [r]}] (* 3 r r))
> (defmulti area-of (fn [shape] (:kind shape)))    ; silently ignored
>
> (area-of {:kind :circle :r 1})
> ;; Throws: No method in multimethod 'scratch.core/area-of' for dispatch value:
> ```
>
> The old dispatch function looked up `:shape`, found `nil`, and failed. When you change a dispatch function, unbind the var first, with `(ns-unmap *ns* 'area-of)` at the REPL, or restart.

---

## Multimethods or protocols?

| | Multimethod | Protocol |
|---|---|---|
| Dispatches on | any function of all the arguments | the type of the first argument |
| Works with plain maps | yes, by a key like `:type` | only via records or `extend` on the map type |
| Hierarchies | yes, via `derive` | no |
| Groups several operations | no, one function each | yes, a named set |
| Speed | a function call plus a lookup | close to a direct method call |

Use a **multimethod** when the distinction is in the data: a `:type` key, a combination of fields, a value. Use a **protocol** when the distinction is the kind of thing, and you want a set of operations that a type implements together. When in doubt with plain-map data, it's a multimethod.

---

## Rules of thumb

- Dispatch on a keyword in the data for maps that carry their own type.
- Provide a `:default` method unless an unknown value should crash.
- Namespace your dispatch keywords (`:user/signup`) so plugins can't collide.
- Put `derive` calls at load time and use `prefer-method` to resolve ambiguity.
- After changing a dispatch function, unmap the var or restart. Re-evaluating `defmulti` does nothing.
- Use protocols for type-based polymorphism, multimethods for value-based.
