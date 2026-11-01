# 27 — Time and logging

*After this chapter you can handle `js/Date` without its traps, do date arithmetic that survives daylight saving, format times for a specific time zone, and emit structured logs.*

---

## `js/Date` is what you have

ClojureScript has no date library of its own. The `#inst` literal reads as a JavaScript `Date`:

```clojure
#inst "2024-01-01T10:00:00Z"            ;; => #inst "2024-01-01T10:00:00.000-00:00"
(type #inst "2024-01-01")               ;; => #object[Date]
(inst-ms #inst "1970-01-01T00:00:01Z")  ;; => 1000     milliseconds since the epoch
(js/Date.now)                           ;; current time in ms
```

A `Date` is a single instant, milliseconds since 1970 UTC. Everything about local time, like the hour or the day, is computed from it using the **process's** time zone. The examples below ran with `TZ=UTC` unless noted, and some results change with `TZ`.

> **Coming from imperative languages:** There's no `time.Time` with a location (Go), no `LocalDate` vs `Instant` (Java), and no naive vs aware split (Python). One type, one instant, with local-time methods that use whatever zone the process runs in. Store and compute in UTC, and convert to a zone only when formatting.

---

## The traps

**Months are zero-based. Days are not.**

```clojure
(.getMonth (js/Date. 2024 0 31))   ;; => 0       January
```

**Out-of-range values roll over silently:**

```clojure
(.toISOString (js/Date. 2024 1 30))   ;; => "2024-03-01T00:00:00.000Z"   "February 30th"
```

**The constructor with numbers uses local time.** With `TZ=Europe/Berlin`, `(js/Date. 2024 0 31)` is `"2024-01-30T23:00:00.000Z"`. The same code gives a different instant on a different machine.

**String parsing is inconsistent.** A date-only string is parsed as UTC, while a date-time without an offset is local time:

```clojure
;; TZ=Europe/Berlin
(.toISOString (js/Date. "2024-03-10"))            ;; => "2024-03-10T00:00:00.000Z"   UTC
(.toISOString (js/Date. "2024-03-10T12:00:00"))   ;; => "2024-03-10T11:00:00.000Z"   local!
```

Always include an explicit offset (`Z` or `+02:00`) in timestamps you parse.

**Invalid dates don't throw:**

```clojure
(js/Date. "not a date")                        ;; => #inst "0NaN-NaN-NaNTNaN:NaN:NaN.NaN-00:00"
(js/isNaN (.getTime (js/Date. "not a date")))  ;; => true    how to check
```

**Dates are mutable:**

```clojure
(let [x (js/Date. 0)]
  (.setFullYear x 2000)
  x)       ;; => #inst "2000-01-01T00:00:00.000-00:00"
```

> ⚠️ **Gotcha:** A `Date` inside an immutable map is still mutable. Anyone holding the map can call a setter and change it for everyone. Never mutate a `Date` you didn't just create: copy it first with `(js/Date. (.getTime d))`, or store epoch milliseconds instead.

On the plus side, ClojureScript compares dates by value:

```clojure
(= (js/Date. 0) (js/Date. 0))         ;; => true
(compare (js/Date. 0) (js/Date. 1))   ;; => -1    so sort and sort-by work
```

---

## Arithmetic

Durations in milliseconds are exact, so adding an elapsed duration is simple:

```clojure
(defn add-ms [^js date ms] (js/Date. (+ (.getTime date) ms)))
```

Adding *days* is different, because a calendar day isn't always 24 hours. Across a daylight-saving change, in `TZ=Europe/Berlin` where clocks moved forward on 31 March 2024:

```clojure
(defn add-days [^js date n]
  (js/Date. (+ (.getTime date) (* n 24 60 60 1000))))

(let [start (js/Date. 2024 2 31)]      ; local midnight, 31 March
  (.toString (add-days start 1))
  ;; => "Mon Apr 01 2024 01:00:00 GMT+0200"     24 hours later: 1 am, not midnight

  (.toString (doto (js/Date. (.getTime start))
               (.setDate (inc (.getDate start))))))
  ;; => "Mon Apr 01 2024 00:00:00 GMT+0200"     the next calendar day
```

Use milliseconds for elapsed time (timeouts, durations, TTLs), and the calendar setters on a **copy** (`setDate`, or `setUTCDate` for UTC calendar days) for "the same time tomorrow".

For measuring how long something took, use `js/performance.now`, which is monotonic and immune to clock adjustments, not `Date.now`.

---

## Formatting with `Intl`

`Intl.DateTimeFormat` is built into Node and formats an instant for any time zone and locale, independent of the process's `TZ`:

```clojure
(def fmt (js/Intl.DateTimeFormat. "en-GB" #js {:timeZone  "Europe/Paris"
                                                :dateStyle "medium"
                                                :timeStyle "short"}))

(.format fmt #inst "2024-07-01T12:00:00Z")   ;; => "1 Jul 2024, 14:00"

(.format (js/Intl.DateTimeFormat. "en-US" #js {:timeZone "America/New_York"
                                               :hour "numeric" :minute "2-digit"})
         #inst "2024-07-01T12:00:00Z")       ;; => "8:00 AM"
```

Create formatters once and reuse them, since construction is relatively expensive. For machine-readable output, `(.toISOString d)` gives UTC ISO 8601.

The process's zone: `(.. (js/Intl.DateTimeFormat.) resolvedOptions -timeZone)`, for example `"UTC"`.

The Closure Library's `goog.date` has date classes and parsers, such as `goog.date.DateTime.fromRfc822String`. `Intl` covers formatting better, so reach for `goog.date` only for its parsing helpers.

---

## Logging

`println` and `prn` write to stdout, and `js/console.error` writes to stderr. That's enough for a CLI. A service should emit **structured** logs: one JSON object per line, which log collectors index:

```clojure
(def levels {:debug 10 :info 20 :warn 30 :error 40})
(defonce min-level (atom :info))

(defn log [level msg & {:as data}]
  (when (>= (levels level) (levels @min-level))
    (let [entry (merge {:ts    (.toISOString (js/Date.))
                        :level (name level)
                        :msg   msg}
                       data)]
      (.write (if (#{:warn :error} level) js/process.stderr js/process.stdout)
              (str (js/JSON.stringify (clj->js entry)) "\n")))))
```

```clojure
(log :info "server started" :port 8080)
;; stdout: {"ts":"…","level":"info","msg":"server started","port":8080}

(log :debug "not shown")      ; below min-level

(try (throw (ex-info "db down" {:host "db1"}))
     (catch :default e
       (log :error "request failed" :error (ex-message e) :error-data (ex-data e))))
;; stderr: {"ts":"…","level":"error","msg":"request failed","error":"db down","error-data":{"host":"db1"}}
```

Notes on the design:

- **Data, not interpolated strings.** `:path "/users" :ms 1200` can be queried, while "slow request to /users took 1200ms" can only be grepped.
- **`ex-data` belongs in the log entry.** It's the most useful context an error has, and Node's default output drops it (chapter 10).
- **The level lives in an atom**, so it can be changed at runtime from the REPL, or from an env var at startup.
- **Writes go straight to the streams.** Leave rotation, shipping and retention to the environment (systemd, a container runtime, a log agent).

Mind what `clj->js` does to the entry: keywords become strings and namespaces are dropped (chapter 23). Keep log data to strings, numbers and simple maps.

---

## Rules of thumb

- Store and compute in UTC. Convert to a zone only when formatting.
- Parse only timestamps with explicit offsets, and check for invalid dates with `isNaN` on `.getTime`.
- Never mutate a `Date` you didn't just create. Copy, or keep epoch milliseconds.
- Use milliseconds for elapsed time and calendar setters for calendar days.
- Format with a reused `Intl.DateTimeFormat` and an explicit `timeZone`.
- Measure durations with `performance.now`.
- Log one JSON object per line, with data as fields and `ex-data` included.
