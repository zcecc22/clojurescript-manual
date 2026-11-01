# 20 — Testing

*After this chapter you can write and run tests with `cljs.test` and shadow-cljs's `:node-test` target, test async code without false passes, and use fixtures.*

---

## Setup

Tests live in a separate source directory with namespaces ending in `-test`. Add the directory and a test build to `shadow-cljs.edn`:

```clojure
{:source-paths ["src" "test"]
 :builds
 {:app  {:target :node-script :main app.main/main :output-to "out/app.js"}
  :test {:target    :node-test
         :output-to "out/test.js"
         :ns-regexp "-test$"}}}
```

`:node-test` finds every namespace matching `:ns-regexp`, generates a runner, and compiles it to one script:

```sh
npx shadow-cljs compile test && node out/test.js
```

The process exits with code 1 if anything fails, which is what CI needs. During development, `npx shadow-cljs watch test` with `:autorun true` in the build config reruns the tests on every save.

> ⚠️ **Gotcha:** A running `shadow-cljs server` doesn't pick up changes to `:source-paths`. After adding `"test"`, the build ran **0 tests** and reported success until the server was restarted. If a new test namespace is ignored, restart the server.

---

## Writing tests

The code under test:

```clojure
(ns app.cart)

(defn add-item [cart item]
  (update cart :items (fnil conj []) item))

(defn total [{:keys [items]}]
  (transduce (map #(* (:price %) (:qty %))) + 0 items))
```

The tests, in `test/app/cart_test.cljs`:

```clojure
(ns app.cart-test
  (:require [cljs.test :refer [deftest is testing are async use-fixtures]]
            [app.cart :as cart]))

(deftest add-item-test
  (testing "first item creates the vector"
    (is (= {:items [{:sku "a"}]} (cart/add-item {} {:sku "a"}))))
  (testing "items keep order"
    (is (= ["a" "b"]
           (->> (-> {} (cart/add-item {:sku "a"}) (cart/add-item {:sku "b"}))
                :items
                (map :sku))))))
```

- `deftest` defines a test, which is just a function with no arguments.
- `is` asserts that an expression is truthy. Put the expected value first: `(is (= expected actual))`.
- `testing` labels a group of assertions in failure output.

A failing assertion reports the expression and the evaluated values:

```clojure
(deftest failing-test
  (is (= 10 (cart/total {:items [{:price 2 :qty 3}]})) "deliberately wrong"))
```

```
FAIL in (failing-test) (app/cart_test.cljs:25:7)
deliberately wrong
expected: (= 10 (cart/total {:items [{:price 2, :qty 3}]}))
  actual: (not (= 10 6))

Ran 6 tests containing 10 assertions.
1 failures, 0 errors.
```

Value equality on whole data structures (chapter 02) makes assertions short. You compare the entire expected map, not field by field.

### Table tests with `are`

```clojure
(deftest total-test
  (are [items expected] (= expected (cart/total {:items items}))
    []                                    0
    [{:price 2 :qty 3}]                   6
    [{:price 2 :qty 3} {:price 1 :qty 1}] 7))
```

`are` takes a binding vector, a template assertion, and rows of values. It's the equivalent of Go's table-driven tests, without the loop.

### Exceptions

```clojure
(deftest throws-test
  (is (thrown? js/Error (throw (js/Error. "x"))))
  (is (thrown-with-msg? js/Error #"unknown" (throw (ex-info "unknown sku" {})))))
```

To assert on the error *data*, catch it yourself and compare `ex-data`.

---

## Async tests

Tests are synchronous by default: when the body returns, the test is over. For async code, wrap the body in `async` and call `done` when you're finished:

```clojure
(deftest async-test
  (async done
    (-> (cart/fetch-price "abc")
        (.then (fn [p] (is (= 9.99 p))))
        (.catch (fn [e] (is false (str "unexpected: " e))))
        (.finally done))))
```

The test that something *rejects* inverts the branches:

```clojure
(deftest async-reject-test
  (async done
    (-> (cart/fetch-price "missing")
        (.then (fn [_] (is false "should have rejected")))
        (.catch (fn [e] (is (= {:sku "missing"} (ex-data e)))))
        (.finally done))))
```

Calling `done` in `.finally` guarantees it runs exactly once, whichever branch ran.

> ⚠️ **Gotcha:** Without a `.catch`, a rejection isn't a test failure. It's an unhandled rejection, which **crashes the entire test process** (chapter 12) partway through the run. Every async test needs both branches and a `.finally done`.

> ⚠️ **Gotcha:** An assertion in a callback *without* `async` runs after the test has already finished. It either attaches to the wrong test or vanishes, and the test passes regardless. If a test touches a promise, it needs `async`.

---

## Fixtures

`use-fixtures` runs setup and teardown around each test (`:each`) or around the whole namespace (`:once`). The map form takes `:before` and `:after` functions:

```clojure
(def log (atom []))

(use-fixtures :each
  {:before (fn [] (reset! log []))
   :after  (fn [] (swap! log conj :after))})

(deftest fixture-a (swap! log conj :a) (is (= [:a] @log)))
(deftest fixture-b (swap! log conj :b) (is (= [:b] @log)))   ; passes: state was reset
```

Fixtures suit resetting shared state like an in-memory store, or creating and cleaning a temporary directory. With pure functions (chapter 11) most tests need none.

---

## Running a subset

The generated runner accepts a filter:

```sh
node out/test.js --list                              # list test names
node out/test.js --test=app.cart-test                # one namespace
node out/test.js --test=app.cart-test/total-test     # one test
```

At the REPL, `(cljs.test/run-tests 'app.cart-test)` runs a namespace, and calling a test like a function, `(total-test)`, runs one.

---

## What to test

The structure the earlier chapters recommend, pure functions in the middle and effects at the edges, pays off here:

- **Pure functions** take data and return data. Test them directly, with no setup and no mocks. That's most of your logic.
- **Effectful edges** are thin. Test them through async tests against real resources where that's cheap: a temp directory, a server on port 0.
- **Spec instrumentation** (chapter 19) in the test build catches calls with malformed arguments. Call `stest/instrument` at the top of a test namespace or in a `:once` fixture.

Where you do need to replace a dependency, `with-redefs` temporarily rebinds a var. It only lasts during the synchronous body, so it has the same async limitation as `binding` (chapter 11). For async code, pass dependencies in as arguments instead.

---

## Rules of thumb

- Test namespaces end in `-test` and live under `test/`.
- Write `(is (= expected actual))` and compare whole data structures.
- Use `are` for table tests.
- Every async test uses `async` with both branches handled and `done` in `.finally`.
- Use fixtures for shared state, and design code so most tests need none.
- Restart the shadow-cljs server after changing `:source-paths`.
