# 10 — Errors

*After this chapter you can throw errors that carry data, catch them by type, chain causes, and decide when an error should be an exception and when it should be a value.*

---

## `ex-info`: errors with data

The idiomatic error is an `ex-info`: a message plus a map of data:

```clojure
(defn withdraw [balance amount]
  (if (> amount balance)
    (throw (ex-info "Insufficient funds"
                    {:type :insufficient-funds :balance balance :amount amount}))
    (- balance amount)))

(try
  (withdraw 10 50)
  (catch :default e
    [(ex-message e) (ex-data e)]))
;; => ["Insufficient funds" {:type :insufficient-funds, :balance 10, :amount 50}]
```

The data map is where the useful information goes: what failed, with what inputs, and a `:type` keyword the caller can branch on. Keep the message for humans and the data for code.

An `ex-info` is a real JavaScript `Error` (its type is `ExceptionInfo`, a subclass), so it has a stack trace and works with any JS code that expects errors:

```clojure
(instance? js/Error (ex-info "x" {}))   ;; => true
(ex-info "msg" {:a 1})                  ;; => #error {:message "msg", :data {:a 1}}
```

> **Coming from imperative languages:** This is the equivalent of a custom error type with fields, without declaring a type. Go's `errors.As` into a struct becomes `(:type (ex-data e))`. Java's exception hierarchy becomes a keyword in a map.

---

## `try`, `catch`, `finally`

`try` is an expression. It returns the body's value, or the matching `catch` clause's value:

```clojure
(try
  (withdraw 100 50)
  (catch :default e :failed)
  (finally (println "finally runs")))
;; prints: finally runs
;; => 50
```

`finally` runs whatever happened, and its value is discarded.

`catch` takes a type and a binding. Clauses are tried in order, and `:default` catches **anything**:

```clojure
(try
  (throw (js/TypeError. "bad type"))
  (catch js/TypeError e [:type-error (.-message e)])
  (catch :default e :other))
;; => [:type-error "bad type"]
```

Catching `js/Error` catches every `Error` subclass, `ex-info` included. Use `:default` as the final clause when you mean "catch everything".

### Branching on error data

Since most of your errors are `ex-info`, you branch on the data, not the type:

```clojure
(defn try-withdraw [balance amount]
  (try
    (withdraw balance amount)
    (catch :default e
      (case (:type (ex-data e))
        :insufficient-funds balance   ; ours: leave the balance unchanged
        (throw e)))))                 ; not ours: rethrow

(try-withdraw 10 50)    ;; => 10
(try-withdraw 100 30)   ;; => 70
```

Rethrow what you don't handle. A `catch :default` that swallows everything turns bugs into silent wrong answers.

---

## Anything can be thrown

JavaScript lets you throw any value, and so does ClojureScript:

```clojure
(try (throw "a string")
  (catch js/Error e :error)
  (catch :default e [:default e (ex-message e)]))
;; => [:default "a string" nil]

(try (throw {:code 1}) (catch :default e e))   ;; => {:code 1}
```

A thrown non-Error has no stack trace, and `catch js/Error` won't catch it. Only `:default` does.

> ⚠️ **Gotcha:** JavaScript libraries occasionally throw strings or plain objects, and so do some Node APIs. If your error handling must not let anything escape, end with `catch :default`, and don't assume the caught value is an `Error`: `ex-message` and `ex-data` return `nil` on anything else.

Always throw `ex-info` or a `js/Error` yourself. Throwing a map or keyword loses the stack trace.

---

## Causes

Wrap a low-level error in a higher-level one by passing it as the third argument to `ex-info`. `ex-cause` gets it back:

```clojure
(try
  (try
    (js/JSON.parse "{bad")
    (catch :default cause
      (throw (ex-info "config unreadable" {:file "x.json"} cause))))
  (catch :default e
    [(ex-message e) (ex-data e) (ex-message (ex-cause e))]))
;; => ["config unreadable" {:file "x.json"} "Expected property name or '}' in JSON at position 1"]
```

Each layer adds the context it knows: which file, which user, which step. The original error stays reachable at the bottom.

Native JavaScript errors carry their cause in `.-cause`: `(js/Error. "msg" #js {:cause root})`.

---

## Errors you'll actually see

Most runtime errors in ClojureScript are JavaScript `TypeError`s from `nil` reaching JS code, and protocol errors from passing the wrong kind of value to a core function:

```clojure
((fn [x] (.toUpperCase x)) nil)
;; Throws: Cannot read properties of null (reading 'toUpperCase')

(assoc 1 :a 1)
;; Throws: No protocol method IAssociative.-assoc defined for type number: 1
```

The "No protocol method" message is the one that tells you a value of the wrong type got somewhere. Read the type at the end of the message.

Core functions mostly don't throw on `nil` (chapter 03): `(:k nil)` is `nil`. So the error tends to surface far from where the `nil` came from, usually at the first JavaScript interop call.

---

## Uncaught errors

An uncaught error, including one thrown from a callback, crashes the Node process with exit code 1 and prints the stack:

```clojure
(js/setTimeout #(throw (ex-info "late" {:a 1})) 0)
;; process exits with code 1, printing:  Error: late  (followed by the stack)
```

Node's default output shows the message and stack but **not** the `ex-data`. Log errors yourself at your top-level boundaries with `prn` or `pr-str`, which show the data, so that it isn't lost.

A `try` only covers code that runs synchronously inside it. An error in a `setTimeout` callback, an event handler or a promise happens later, outside the `try` (chapters 12 and 13).

---

## Exceptions or values?

Not every failure should be an exception. A useful split:

- **Throw** for bugs, broken invariants, and failures the immediate caller can't do anything sensible about: a missing config file at startup, a database that's down.
- **Return a value** for outcomes the caller is expected to handle: "not found", invalid user input, a validation result.

A value can be `nil` for "nothing there", or a map that says what happened:

```clojure
(defn parse-port [s]
  (let [n (parse-long s)]
    (if (and n (< 0 n 65536))
      {:ok n}
      {:error (str "invalid port: " s)})))

(parse-port "8080")   ;; => {:ok 8080}
(parse-port "nope")   ;; => {:error "invalid port: nope"}
```

> **Coming from imperative languages:** Go readers will recognise the second style, and Python and Java readers the first. ClojureScript uses both: exceptions are the default mechanism, as in JavaScript, but returning data for expected outcomes is idiomatic and keeps `try` blocks rare.

---

## Rules of thumb

- Throw `ex-info` with a `:type` keyword and the relevant inputs in the data map.
- Branch on `(:type (ex-data e))`, and rethrow what you don't handle.
- End a catch-everything handler with `:default`, and don't assume what it caught is an `Error`.
- Wrap with a cause rather than replacing: `(ex-info msg data cause)`.
- Log with `prn` at the top level so `ex-data` isn't lost.
- Return values for expected outcomes, throw for the unexpected.
