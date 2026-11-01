# 23 — Data formats

*After this chapter you can move data in and out as JSON and EDN without losing information, pretty-print data for humans, and handle simple delimited text.*

---

## JSON in

`js/JSON.parse` produces JavaScript objects. Convert them to ClojureScript data once, at the boundary:

```clojure
(def payload "{\"id\": 12, \"user_name\": \"ada\", \"roles\": [\"admin\"],
               \"meta\": {\"active\": true, \"score\": null}}")

(js->clj (js/JSON.parse payload) :keywordize-keys true)
;; => {:id 12, :user_name "ada", :roles ["admin"], :meta {:active true, :score nil}}
```

`null` becomes `nil`, arrays become vectors, and objects become maps.

Invalid input throws a `SyntaxError`, so wrap parsing of anything external:

```clojure
(js/JSON.parse "{bad}")
;; Throws: Expected property name or '}' in JSON at position 1
```

> ⚠️ **Gotcha:** `:keywordize-keys` turns **any** string into a keyword, including ones that can't be written as keyword literals:
>
> ```clojure
> (js->clj (js/JSON.parse "{\"user-name\": 1, \"2fa\": true, \"a b\": 3}") :keywordize-keys true)
> ;; => {:user-name 1, :2fa true, :a b 3}
> ```
>
> `:a b` is a keyword containing a space. It works as a map key, but it doesn't print readably and you can't type it. When keys are user-controlled, such as headers, form fields or dictionary-like objects, leave them as strings.

> ⚠️ **Gotcha:** JSON numbers are parsed into doubles before your code sees them (chapter 03):
>
> ```clojure
> (js/JSON.parse "{\"id\": 12345678901234567890}")
> ;; => #js {:id 12345678901234567000}
> ```
>
> The ID is already wrong, and no conversion afterwards can fix it. Large IDs must travel as strings.

---

## JSON out

```clojure
(defn ->json [x] (js/JSON.stringify (clj->js x)))

(->json {:id 1 :status :active :tags #{:a} :at (js/Date. 0) :n nil})
;; => "{\"id\":1,\"status\":\"active\",\"tags\":[\"a\"],\"at\":\"1970-01-01T00:00:00.000Z\",\"n\":null}"
```

Pretty-print with the third argument:

```clojure
(println (js/JSON.stringify (clj->js {:a 1 :b [1 2]}) nil 2))
```

### JSON is lossy for ClojureScript data

The round trip doesn't come back the same:

| ClojureScript | JSON | Back again |
|---|---|---|
| keyword value `:active` | `"active"` | the string `"active"` |
| set `#{:a}` | `["a"]` | a vector |
| `js/Date` | ISO string | a string |
| namespaced key `:user/id` | `"id"` | collides with other `id` keys (chapter 09) |
| function | silently dropped | gone |
| `undefined` | `null` (via `clj->js`) | `nil` |
| `##NaN`, `##Inf` | `null` | `nil` |

```clojure
(js->clj (js/JSON.parse (->json {:status :active})) :keywordize-keys true)
;; => {:status "active"}

(js/JSON.stringify (clj->js {:f (fn [] 1) :u js/undefined :nan ##NaN}))
;; => "{\"u\":null,\"nan\":null}"     :f vanished, undefined became null
```

So at a JSON boundary, convert values explicitly: keywords back from strings for known enum fields, dates parsed from strings. A spec conformer (chapter 19) or a small decoding function per message type keeps this in one place.

### The cost of conversion

`js->clj` and `clj->js` walk the entire structure and allocate a copy. For a few kilobytes that's irrelevant. For large payloads that you only pass through or read a field from, skip the conversion and use interop on the JS object directly, with `goog.object/get` and `.-prop` (chapter 09). Chapter 28 has numbers.

---

## EDN

EDN, Extensible Data Notation, is Clojure's literal syntax as a data format. It round-trips ClojureScript values: keywords, sets, symbols, dates and UUIDs all survive.

Write with `pr-str`, read with `clojure.edn/read-string`:

```clojure
(:require [clojure.edn :as edn])

(def s (pr-str {:id 1 :tags #{:a :b} :at (js/Date. 0)
                :uid (uuid "00000000-0000-0000-0000-000000000001")
                :nested [1 {:x "y"}] :sym 'foo}))

s
;; => "{:id 1, :tags #{:b :a}, :at #inst \"1970-01-01T00:00:00.000-00:00\",
;;      :uid #uuid \"00000000-0000-0000-0000-000000000001\", :nested [1 {:x \"y\"}], :sym foo}"

(edn/read-string s)
;; => the same map, with a js/Date and a UUID back
```

Use EDN for config files, for data files your program writes and reads back, and for messages between Clojure programs. Use JSON when anything else consumes the data.

### Reading is safe

`clojure.edn` reads **data only**. It never evaluates anything:

```clojure
(edn/read-string "(+ 1 2)")    ;; => (+ 1 2)   a list, not 3
(edn/read-string "#=(+ 1 2)")  ;; Throws: No reader function for tag =.
(edn/read-string "{:a 1 :a 2}") ;; Throws: Map literal contains duplicate key: :a
```

Empty input returns `nil`, or a value you choose:

```clojure
(edn/read-string "")                 ;; => nil
(edn/read-string {:eof :done} "")    ;; => :done
```

`cljs.reader/read-string` also exists and gives the same results for ordinary data. Prefer `clojure.edn`, whose name says what it's for and matches JVM Clojure.

### Tagged literals

EDN is extensible with tags. `#inst` and `#uuid` are built in. Others need a reader function:

```clojure
(edn/read-string "#money \"12.50\"")
;; Throws: No reader function for tag money.

(edn/read-string {:readers {'money (fn [s] {:amount (parse-double s) :currency :eur})}}
                 "#money \"12.50\"")
;; => {:amount 12.5, :currency :eur}
```

---

## Pretty-printing

`cljs.pprint/pprint` lays out nested data readably:

```clojure
(:require [cljs.pprint :as pprint])

(pprint/pprint {:users [{:id 1 :name "Ada" :roles #{:admin :ops}
                         :address {:city "London" :zip "N1"}} ...]})
;; {:users
;;  [{:id 1,
;;    :name "Ada",
;;    :roles #{:admin :ops},
;;    :address {:city "London", :zip "N1"}}
;;   ...]}
```

`print-table` prints maps as a table:

```clojure
(pprint/print-table [:id :name] [{:id 1 :name "Ada"} {:id 2 :name "Bob"}])
;; | :id | :name |
;; |-----+-------|
;; |   1 |   Ada |
;; |   2 |   Bob |
```

`with-out-str` captures printed output as a string, and `*print-length*` caps how much of a long or infinite sequence prints:

```clojure
(with-out-str (pprint/pprint [1 2]))     ;; => "[1 2]\n"
(binding [*print-length* 3] (prn (range 100)))   ;; prints: (0 1 2 ...)
```

`cljs.pprint` is large. It noticeably increases release bundle size, so keep it to development tools and CLIs that actually print tables.

---

## Delimited text

There's no CSV parser in core. For simple, well-behaved data you control (no quoted fields, no embedded commas or newlines), splitting is enough:

```clojure
(defn parse-csv-line [line]
  (str/split line #"," -1))

(map parse-csv-line (str/split-lines "a,b,c\n1,,3"))
;; => (["a" "b" "c"] ["1" "" "3"])
```

> ⚠️ **Gotcha:** `str/split` drops trailing empty strings unless you pass a limit. `(str/split "a,b,," #",")` is `["a" "b"]`, while `(str/split "a,b,," #"," -1)` is `["a" "b" "" ""]`. For CSV, empty trailing fields are data, so always pass `-1`.

Real CSV, with quoted fields, escaped quotes and newlines inside quotes, needs a proper state-machine parser. That's about forty lines, or a library. Don't stretch `split` to cover it.

---

## Rules of thumb

- Convert JSON at the boundary: `js->clj` with `:keywordize-keys` in, `clj->js` out.
- Keep user-controlled keys as strings.
- Move large integer IDs as strings.
- Decode JSON values explicitly. Keywords, sets and dates don't survive JSON.
- Use EDN for config and for data your program reads back, and read it with `clojure.edn`.
- Register tagged-literal readers for your own types.
- Pass `-1` to `str/split` when empty trailing fields matter.
