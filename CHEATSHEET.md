# ClojureScript Cheatsheet

Fast lookup. Each section links to the chapter that explains it.

---

## Project · [Ch. 01](docs/01-orientation.md) · [Ch. 21](docs/21-tooling.md)

```clojure
;; shadow-cljs.edn
{:source-paths ["src" "test"]
 :builds {:app  {:target :node-script :main app.main/main :output-to "out/app.js"
                 :compiler-options {:warnings-as-errors true}}
          :test {:target :node-test :output-to "out/test.js" :ns-regexp "-test$"}}}
```

```sh
npx shadow-cljs server            # keep running: later commands are fast
npx shadow-cljs watch app         # dev build + hot reload
npx shadow-cljs node-repl         # REPL in a Node process
npx shadow-cljs release app       # optimised single file
npx shadow-cljs release app --debug   # optimised, readable names
npx shadow-cljs compile test && node out/test.js
```

`my-app.core` lives at `src/my_app/core.cljs`.

---

## Literals and truthiness · [Ch. 02](docs/02-syntax-and-evaluation.md) · [Ch. 03](docs/03-values-and-types.md)

```clojure
42  3.14  1e3  "str"  \a          ; \a is the string "a"
:kw  :ns/kw  ::local  'sym
[1 2]  '(1 2)  {:a 1}  #{1 2}  #"re+"
nil  true  false
#js {:a 1}  #js [1 2]             ; shallow
#inst "2024-01-01T00:00:00Z"  #uuid "..."
#_ (ignored form)  ; comment  (comment ...)
```

**Falsy: only `nil` and `false`.** `0`, `""`, `[]` and `##NaN` are truthy.

```clojure
(= {:a 1 :b 2} {:b 2 :a 1})  ;; => true   value equality, deep
(= [1 2] '(1 2))             ;; => true
(= 1 1.0)                    ;; => true   one number type: double
(identical? :a :a)           ;; => false  keywords aren't interned: use =
(/ 10 4)                     ;; => 2.5    (quot 10 4) => 2
(mod -7 3) (rem -7 3)        ;; => 2  -1
(parse-long "42px")          ;; => nil
```

---

## Functions · [Ch. 04](docs/04-functions.md)

```clojure
(defn greet
  "Docstring."
  ([] (greet "world"))                  ; multi-arity = defaults
  ([name] (str "Hello, " name)))

(defn sum [& xs] (reduce + 0 xs))       ; variadic; xs is nil when empty
(apply sum [1 2 3])
(fn [x] (* 2 x))   #(* 2 %)   #(+ %1 %2)   #(vector %&)

(defn area [{:keys [w h] :or {h 1} :as rect}] (* w h))   ; map destructuring
(let [[a b & more] [1 2 3 4]] ...)                         ; sequential

(partial + 10)  (comp str inc)  (juxt :a :b)  (complement even?)
(constantly 7)  identity  (memoize f)  (fnil inc 0)
```

```clojure
(-> m (assoc :b 2) (dissoc :a))         ; value as FIRST arg: collections
(->> xs (filter even?) (map inc))       ; value as LAST arg: sequences
(some-> m :a :b inc)                    ; stop at nil
(cond-> m flag (assoc :x 1))
(as-> 5 x (* x 2) (- 100 x))
```

---

## Control flow · [Ch. 05](docs/05-control-flow.md)

```clojure
(if test then else)        (when test & body)     (when-not ...)
(if-let [x (f)] x alt)     (when-let [x (f)] ...)  (if-some [x (f)] ...)
(cond (neg? n) :neg (zero? n) :zero :else :pos)
(case code 200 "OK" (500 503) "error" "default")   ; literals only; no default throws
(condp = x 1 :one 2 :two :other)
(let [x 1 x (inc x)] x)    (do a b c)

(loop [i 0 acc []] (if (< i 3) (recur (inc i) (conj acc i)) acc))
(doseq [x xs y ys] (println x y))       (dotimes [i 3] ...)
(some pred xs)  (reduce (fn [a x] (if done? (reduced a) ...)) init xs)
```

No `return`, no `break`. `recur` only in tail position. No TCO otherwise (about 8,000 frames).

---

## Collections · [Ch. 06](docs/06-collections.md)

```clojure
;; vector                          ;; map
(conj [1 2] 3)    ;; [1 2 3]       (assoc m :k v)     (dissoc m :k)
(nth v 0) (get v 9) (v 0)          (update m :n inc)  (update m :n + 10)
(assoc v 1 :x) (peek v) (pop v)    (get m :k :default)  (:k m)  (m :k)
(subvec v 1)                       (contains? m :k)   (find m :k)
                                   (merge a b) (merge-with + a b)
;; list: conj at FRONT             (select-keys m [:a]) (zipmap ks vs)
(conj '(1 2) 0)   ;; (0 1 2)       (keys m) (vals m) (update-vals m f)

;; set                             ;; nested
(conj s x) (disj s x)              (get-in m [:a :b])
(contains? s x) (s x)              (assoc-in m [:a :b] v)
(set/union a b) (set/difference a b)   (update-in m [:a :b] f)

(sorted-map ...) (sorted-set ...) #queue []
(contains? [10 20] 1)   ;; => true: checks the INDEX. Use (some #{20} v)
(update {} :xs conj 1)  ;; => {:xs (1)}: a list. Use (fnil conj [])
```

Maps of 9 or more entries lose insertion order. Never rely on map order.

---

## Sequences · [Ch. 07](docs/07-sequences.md) · [Ch. 18](docs/18-transducers.md)

```clojure
(map f xs) (filter p xs) (remove p xs) (keep f xs) (mapcat f xs)
(reduce f init xs) (reduce-kv f init m)
(take n xs) (drop n xs) (take-while p xs) (partition-all n xs)
(group-by :k xs) (frequencies xs) (sort-by :k > xs) (distinct xs)
(some p xs) (every? p xs) (first xs) (rest xs) (next xs) (last xs)
(mapv f xs) (filterv p xs) (vec xs) (into [] xs) (into {} pairs)
(for [x xs :when (odd? x) :let [y (* x x)]] y)       ; lazy comprehension
(range) (iterate f x) (cycle xs) (repeat n x)
(run! f xs) (doall xs) (dorun xs)
```

Lazy and chunked: realised 32 at a time. No side effects inside `map`, `filter` or `for`.

```clojure
(def xf (comp (filter odd?) (map inc) (take 3)))     ; left to right
(into [] xf xs)   (transduce xf + xs)   (sequence xf xs)
```

---

## Namespaces · [Ch. 08](docs/08-namespaces.md)

```clojure
(ns app.main
  (:require [app.util :as util]
            [clojure.string :as str]
            [cljs.test :refer [deftest is]]
            [goog.object :as gobj]                ; Closure: symbol
            ["node:fs/promises" :as fsp]          ; Node: string, node: prefix
            ["node:events" :refer [EventEmitter]]) ; never :default on built-ins
  (:import [goog Uri]))                           ; Closure classes
```

`.cljc` plus `#?(:clj ... :cljs ...)` for shared code. `(declare f)` for forward references. `defn-` isn't enforced.

---

## JS interop · [Ch. 09](docs/09-js-interop.md)

```clojure
(.method obj a b)          (.-prop obj)          (.. obj -a -b (m x))
(js/Date. 0)  (new js/Date 0)                    (set! (.-x obj) 1)
js/process.env             (js/parseInt "42")    (instance? js/Date d)
(clj->js {:a [1]})         (js->clj o :keywordize-keys true)
(gobj/get o "k")           (aget arr 0)          (this-as this ...)
(defn f [^js x] (.-userName x))                  ; hint for advanced compilation
(.from Readable xs)                              ; static methods: not (Readable.from xs)
```

`(inc "1")` is `"11"`. `clj->js` drops keyword namespaces. `js/console.log` mangles ClojureScript maps, so use `prn`.

---

## Errors · [Ch. 10](docs/10-errors.md)

```clojure
(throw (ex-info "msg" {:type :not-found :id id} cause))
(try ... (catch js/TypeError e ...) (catch :default e ...) (finally ...))
(ex-message e) (ex-data e) (ex-cause e)
```

---

## State · [Ch. 11](docs/11-state.md)

```clojure
(defonce state (atom {}))
@state  (swap! state assoc :k v)  (reset! state {})  (swap-vals! a f)
(add-watch a :key (fn [k ref old new] ...))  (atom 0 :validator pos?)
(volatile! 0) (vswap! v inc)   (delay expr) @d
(def ^:dynamic *x* 1) (binding [*x* 2] ...)  ; sync only
```

---

## Async · [Ch. 12](docs/12-promises-and-async.md) · [Ch. 14](docs/14-async-patterns.md)

```clojure
(defn ^:async f [] (let [x (await (g))] ...))       ; returns a Promise
(fn ^:async named [] ...)  (^:async fn [] ...)      ; anonymous async
(try (await p) (catch :default e ...))
(-> p (.then f) (.catch h) (.finally g))
(js/Promise.all (into-array (map fetch-user ids)))
(js/Promise. (fn [resolve reject] ...))
(util/promisify cb-fn)
#js {:signal (js/AbortSignal.timeout 5000)}         ; cancelling timeout
```

No `await` inside nested `fn`s. Unhandled rejection means the process exits with 1.

---

## Events and streams · [Ch. 13](docs/13-events-and-streams.md)

```clojure
(.on em "evt" f)  (.once em "evt" f)  (.off em "evt" f)  (.emit em "evt" x)
(.on em "error" f)                    ; or "error" throws
(await (events/once em "ready"))
(await (sp/pipeline src transform dst))      ; never .pipe
;; for await: write iterate-async once (chapter 13)
```

---

## Polymorphism · [Ch. 15](docs/15-protocols-and-records.md) · [Ch. 16](docs/16-multimethods.md)

```clojure
(defprotocol Shape (area [this]))
(defrecord Circle [r] Shape (area [_] (* js/Math.PI r r)))
(->Circle 2) (map->Circle {:r 2})
(extend-protocol Shape nil (area [_] 0) number (area [n] (* n n)))
(reify Shape (area [_] 42))

(defmulti handle :type)
(defmethod handle :user/signup [e] ...)
(defmethod handle :default [e] ...)
(derive ::cat ::animal) (prefer-method m ::a ::b)
```

---

## Macros · [Ch. 17](docs/17-macros.md)

```clojure
;; src/app/macros.clj — runs on the JVM
(defmacro unless [test & body] `(if ~test nil (do ~@body)))
(defmacro sq [x] `(let [v# ~x] (* v# v#)))           ; gensym; evaluate once
;; src/app/macros.cljs
(ns app.macros (:require-macros [app.macros]))
(macroexpand-1 '(unless x y))
```

---

## Spec · [Ch. 19](docs/19-spec.md)

```clojure
(s/def ::id pos-int?)
(s/def ::user (s/keys :req-un [::id ::email] :opt-un [::role]))
(s/valid? ::user m) (s/explain-str ::user m) (s/explain-data ::user m)
(s/conform (s/or :id int? :name string?) x)   ;; => [:id 5]
(s/fdef f :args (s/cat :x int?)) (stest/instrument `f)
```

---

## Testing · [Ch. 20](docs/20-testing.md)

```clojure
(deftest t (testing "label" (is (= expected actual))))
(are [x y] (= y (f x))  1 2  3 4)
(is (thrown? js/Error (f)))
(deftest async-t
  (async done (-> (p) (.then #(is ...)) (.catch #(is false %)) (.finally done))))
(use-fixtures :each {:before f :after g})
```

`node out/test.js --test=app.cart-test/total-test`

---

## Node I/O · [Ch. 22](docs/22-files-and-io.md) · [Ch. 26](docs/26-cli-tools.md)

```clojure
(await (fsp/readFile f "utf8"))  (await (fsp/writeFile f s))
(await (fsp/mkdir d #js {:recursive true}))  (await (fsp/rm d #js {:recursive true :force true}))
(let [^js st (await (fsp/stat f))] (.-size st))
(catch :default ^js e (.-code e))                 ; "ENOENT"
(path/join a b) (path/resolve p) (js/Buffer.from s) (.toString buf "base64")
(set! (.-exitCode js/process) 1)                  ; not process.exit after output
(util/parseArgs (clj->js {...}))                  ; Object.assign .values before js->clj
(util/promisify cp/execFile)                      ; not exec
```

---

## HTTP · [Ch. 24](docs/24-http-servers.md) · [Ch. 25](docs/25-http-clients.md)

```clojure
;; server: request map -> response map
(def app (-> router wrap-errors wrap-logging))
{:status 200 :headers {"content-type" "application/json"} :body "..."}
(.listen server 0 cb)  (.-port (.address server))  (.close server cb)

;; client
(let [^js res (await (js/fetch url #js {:signal (js/AbortSignal.timeout 5000)}))]
  (when-not (.-ok res) (throw (ex-info "HTTP" {:status (.-status res)})))
  (await (.json res)))
```

---

## Data formats · [Ch. 23](docs/23-data-formats.md)

```clojure
(js->clj (js/JSON.parse s) :keywordize-keys true)   (js/JSON.stringify (clj->js m))
(pr-str data)  (edn/read-string s)  (edn/read-string {:readers {'tag f}} s)
(pprint/pprint data)  (pprint/print-table rows)
(str/split line #"," -1)
```

---

## Time, crypto · [Ch. 27](docs/27-time-and-logging.md) · [Ch. 31](docs/31-crypto-and-randomness.md)

```clojure
(js/Date.now) (.toISOString d) (js/performance.now)       ; months are 0-based
(.format (js/Intl.DateTimeFormat. "en-GB" #js {:timeZone "Europe/Paris" :timeStyle "short"}) d)
(.toString (crypto/randomBytes 32) "base64url")  (crypto/randomUUID)
(-> (crypto/createHmac "sha256" k) (.update msg) (.digest "hex"))
(crypto/timingSafeEqual a b)        ; equal lengths only
```

`rand`, `shuffle` and `random-uuid` use `Math.random`, so they're not for secrets.
