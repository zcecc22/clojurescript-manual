# 19 — Spec

*After this chapter you can describe the shape of your data with `cljs.spec.alpha`, validate it at the boundaries with readable errors, parse structured input with `conform`, and check function calls in development with instrumentation.*

---

## Why spec

ClojureScript has no static types, and plain maps carry no schema. Spec fills part of that gap: you describe data with predicates, and then validate it, explain failures, parse it, and check function arguments in development. It ships with ClojureScript:

```clojure
(ns app.user
  (:require [cljs.spec.alpha :as s]))
```

Despite the `alpha` in the name, `cljs.spec.alpha` is stable and widely used. The name reflects that its successor has been in design for years.

> **Coming from imperative languages:** Spec isn't a type system. It doesn't run at compile time and doesn't catch mistakes you don't exercise. It's closer to a validation library such as Zod, Pydantic or Go struct tags with a validator, but built from ordinary predicates and integrated with the language. Use it where data crosses a boundary: HTTP input, config files, messages, the database.

---

## Defining specs

Specs are registered under namespaced keywords. Any predicate function is a spec:

```clojure
(s/def ::id pos-int?)
(s/def ::email (s/and string? #(re-matches #".+@.+\..+" %)))
(s/def ::role #{:admin :user})                       ; a set is a predicate
(s/def ::tags (s/coll-of keyword? :kind set?))
```

`s/keys` specs a map by which keys it contains. The value spec for each key is looked up by the key's name:

```clojure
(s/def ::user (s/keys :req-un [::id ::email]
                      :opt-un [::role ::tags]))
```

`:req-un` and `:opt-un` match **unqualified** keys (`:id`) against the specs `::id` and so on. `:req` and `:opt` match **qualified** keys like `:order/id`:

```clojure
(s/def :order/id string?)
(s/def ::order (s/keys :req [:order/id]))

(s/valid? ::order {:order/id "x"})   ;; => true
(s/valid? ::order {:id "x"})         ;; => false
```

> ⚠️ **Gotcha:** `s/keys` maps are **open**. Extra keys are allowed and not checked, so `(s/valid? ::user {:id 1 :email "a@b.co" :extra 1})` is `true`. That's by design: it lets data accumulate as it flows through a system. If you need to reject unknown keys, do it explicitly with `select-keys` or a custom predicate.

---

## Validating and explaining

```clojure
(s/valid? ::user {:id 1 :email "a@b.co"})   ;; => true
(s/valid? ::user {:id 0 :email "nope"})     ;; => false
```

`explain-str` says what's wrong, with the path to each problem:

```clojure
(s/explain-str ::user {:id 0 :email "nope"})
;; 0 - failed: pos-int? in: [:id] at: [:id] spec: :scratch.core/id
;; "nope" - failed: (re-matches #".+@.+\..+" %) in: [:email] at: [:email] spec: :scratch.core/email

(s/explain-str ::user {:email "a@b.co"})
;; {:email "a@b.co"} - failed: (contains? % :id) spec: :scratch.core/user
```

`explain-data` gives the same information as data, which is what you want for an API's error response:

```clojure
(::s/problems (s/explain-data ::user {:id 0 :email "a@b.co"}))
;; => ({:path [:id], :pred cljs.core/pos-int?, :val 0,
;;      :via [:scratch.core/user :scratch.core/id], :in [:id]})
```

A typical boundary check:

```clojure
(defn parse-user! [m]
  (if (s/valid? ::user m)
    m
    (throw (ex-info "invalid user" {:type     :validation
                                    :problems (::s/problems (s/explain-data ::user m))}))))
```

---

## Collections

```clojure
(s/coll-of int? :min-count 2)        ; also :kind, :count, :max-count, :distinct
(s/map-of keyword? number?)          ; homogeneous maps
(s/tuple int? string?)               ; fixed-position vectors
(s/nilable string?)                  ; nil or a string
```

```clojure
(s/valid? (s/map-of keyword? number?) {:a 1})   ;; => true
(s/valid? (s/coll-of int? :min-count 2) [1])    ;; => false
```

---

## `conform`: validating and parsing

`conform` returns the data, transformed to show **how** it matched, or `::s/invalid`. `s/or` tags which branch matched:

```clojure
(s/def ::id-or-name (s/or :id pos-int? :name string?))

(s/conform ::id-or-name 5)       ;; => [:id 5]
(s/conform ::id-or-name "ada")   ;; => [:name "ada"]
(s/conform ::id-or-name :x)      ;; => :cljs.spec.alpha/invalid
```

Regex specs (`s/cat`, `s/*`, `s/+`, `s/?`, `s/alt`) describe sequences and conform them to maps. That's a small parser for argument lists, command vectors or DSLs:

```clojure
(s/def ::args (s/cat :cmd keyword?
                     :opts (s/* (s/cat :k keyword? :v any?))))

(s/conform ::args [:run :port 80 :host "h"])
;; => {:cmd :run, :opts [{:k :port, :v 80} {:k :host, :v "h"}]}
```

`s/conformer` lets a spec transform values, such as coercing strings:

```clojure
(s/def ::celsius
  (s/conformer #(if (number? %) % (or (parse-double (str %)) ::s/invalid))))

(s/conform ::celsius "21.5")   ;; => 21.5
```

Use conformers sparingly. Spec is primarily for validation, and heavy coercion is clearer as a plain function before validation.

---

## Function specs and instrumentation

`s/fdef` specs a function's arguments and return value. `instrument`, from `cljs.spec.test.alpha`, wraps the function so that every call checks its **arguments**:

```clojure
(:require [cljs.spec.test.alpha :as stest])

(defn add-user [user] (assoc user :created true))

(s/fdef add-user
  :args (s/cat :user ::user)
  :ret  map?)

(stest/instrument `add-user)

(add-user {:id 1 :email "a@b.co"})   ;; => {:id 1, :email "a@b.co", :created true}
(add-user {:id -1})
;; Throws: Call to #'scratch.core/add-user did not conform to spec.
;;   (ex-data holds ::s/problems, ::s/args and more)

(stest/unstrument `add-user)
(add-user {:id -1})                  ;; => {:id -1, :created true}   unchecked again
```

Instrument in development and tests, never in production, since it slows every call. Note that `instrument` checks only `:args`. `:ret` and `:fn` are checked by generative testing with `stest/check`, which needs the `test.check` library and is outside this manual's core-only scope.

### Assertions

`s/assert` checks a value and returns it, but only when assertion checking is on. It's off by default, so asserts cost nothing in production:

```clojure
(s/check-asserts true)
(s/assert ::id -5)   ;; Throws: Spec assertion failed  -5 - failed: pos-int?
(s/assert ::id 5)    ;; => 5
```

### Validating state

A spec makes a good atom validator (chapter 11):

```clojure
(s/def ::port (s/and int? #(< 0 % 65536)))
(s/def ::config (s/keys :req-un [::port]))

(def config (atom {:port 80} :validator #(s/valid? ::config %)))
(swap! config assoc :port 0)   ;; Throws: Validator rejected reference state
```

---

## Rules of thumb

- Spec data at the boundaries: input, config, messages, persistence. Not every internal function.
- Register specs under namespaced keywords, one `s/def` per attribute.
- Remember `s/keys` maps are open. Strip unknown keys explicitly if you must reject them.
- Return `explain-data` problems in error responses, and `explain-str` in logs.
- Use `conform` with `s/or` and `s/cat` to parse structured input.
- Instrument in development and tests only. It checks `:args`, not `:ret`.
