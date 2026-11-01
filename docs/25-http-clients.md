# 25 — HTTP clients

*After this chapter you can call HTTP APIs with the built-in `fetch`, treat non-2xx responses as errors deliberately, set timeouts that actually cancel, and distinguish network failures from HTTP failures.*

---

## `fetch` is built in

Node 20 has the WHATWG `fetch` as a global. No require, no library:

```clojure
(defn ^:async get-json [url]
  (let [^js res (await (js/fetch url))]
    (when-not (.-ok res)
      (throw (ex-info (str "HTTP " (.-status res))
                      {:type   :http-error
                       :status (.-status res)
                       :body   (await (.text res))})))
    (js->clj (await (.json res)) :keywordize-keys true)))

(await (get-json (str base "/json")))   ;; => {:id 1, :name "Ada"}
```

Two awaits: `fetch` resolves as soon as the **headers** arrive, and `.json` or `.text` reads the body. The `^js` hint on `res` keeps its property names intact in a release build (chapter 22).

---

## A 404 is not an error

> ⚠️ **Gotcha:** `fetch` only rejects when there's no HTTP response at all. A 404 or a 500 **resolves** normally:
>
> ```clojure
> (let [^js r (await (js/fetch (str base "/missing")))]
>   [(.-status r) (.-ok r)])      ;; => [404 false]
> ```
>
> If you call `.json` on that response without checking, you'll get a parse error about the error page's HTML, far from the real cause. Check `.-ok` (true for 200–299) on every response, as `get-json` does.

With the check in place, an HTTP error becomes a proper `ex-info` the caller can branch on:

```clojure
(try (await (get-json (str base "/missing")))
     (catch :default e [(ex-message e) (ex-data e)]))
;; => ["HTTP 404" {:type :http-error, :status 404, :body "nope"}]
```

---

## Responses

```clojure
(let [^js r (await (js/fetch (str base "/json")))]
  (.-status r)                               ;; => 200
  (.get (.-headers r) "x-request-id")        ;; => "abc"
  (.get (.-headers r) "Content-Type")        ;; => "application/json"   case-insensitive
  (await (.text r)))                         ;; => "{\"id\":1,\"name\":\"Ada\"}"
```

> ⚠️ **Gotcha:** The body can be read **once**. A second `.text` or `.json` on the same response throws `Body is unusable: Body has already been read`. Read it into a value and use the value.

Body readers are `.text`, `.json`, `.arrayBuffer` and `.blob`. `.-body` is a web `ReadableStream` for streaming large downloads.

---

## Requests

POST JSON by setting the method, a content type, and a string body:

```clojure
(defn ^:async post-json [url data]
  (let [^js res (await (js/fetch url
                                 #js {:method  "POST"
                                      :headers #js {"content-type" "application/json"}
                                      :body    (js/JSON.stringify (clj->js data))}))]
    (js->clj (await (.json res)) :keywordize-keys true)))

(await (post-json (str base "/echo") {:name "Bob" :tags [:a]}))
;; the echo server reports => {:method "POST", :ct "application/json", :body {:name "Bob", :tags ["a"]}}
```

The options object is JavaScript, so build it with `#js` (nested `#js` for headers) or `clj->js`.

Build URLs with `js/URL`, which encodes query parameters correctly:

```clojure
(let [u (js/URL. "https://api.example.com/search")]
  (.set (.-searchParams u) "q" "clojure script")
  (.set (.-searchParams u) "page" 2)
  (.toString u))
;; => "https://api.example.com/search?q=clojure+script&page=2"
```

Form bodies use `URLSearchParams`:

```clojure
(.toString (js/URLSearchParams. (clj->js {:a "1" :b "x y"})))   ;; => "a=1&b=x+y"
```

---

## Timeouts

`fetch` has **no default timeout**: a server that accepts the connection and never answers hangs your request forever. Pass a signal:

```clojure
(try
  (await (js/fetch (str base "/slow") #js {:signal (js/AbortSignal.timeout 50)}))
  (catch :default ^js e
    [(.-name e) (ex-message e)]))
;; => ["TimeoutError" "The operation was aborted due to timeout"]
```

This one actually cancels the request and frees the socket, unlike a `Promise.race` (chapter 14). Note the error name: `fetch` with `AbortSignal.timeout` rejects with `TimeoutError`, while a manual `.abort` gives `AbortError`. Check for both if you handle cancellation.

The timeout covers the whole operation, including reading the body, as long as you read it before the signal fires.

---

## Network errors

When there's no response at all, `fetch` rejects with a `TypeError` whose message is just `"fetch failed"`. The useful information is in `.-cause`:

```clojure
(try
  (await (js/fetch "http://localhost:59999"))   ; nothing listening
  (catch :default ^js e
    [(.-name e) (ex-message e) (.. e -cause -code)]))
;; => ["TypeError" "fetch failed" "ECONNREFUSED"]
```

For `localhost` the cause is an `AggregateError`, from one attempt over IPv4 and one over IPv6, and it still carries the `code`. Typical codes are `ECONNREFUSED`, `ENOTFOUND` (DNS), `ECONNRESET` and `UND_ERR_CONNECT_TIMEOUT`.

> ⚠️ **Gotcha:** `fetch` refuses some ports outright, for security. A test against `http://localhost:1` fails with cause `bad port`, not a connection error. Use an ordinary high port when testing failures.

A bad URL is also a `TypeError`, thrown before anything goes on the network: `Failed to parse URL from not a url`.

So there are three kinds of failure, and they want different handling:

| Failure | Shape | Retry? |
|---|---|---|
| Network (refused, DNS, reset) | `TypeError "fetch failed"`, code in `.-cause` | often, with backoff |
| Timeout | `TimeoutError` | sometimes, if the operation is idempotent |
| HTTP error | resolves, `.-ok` false | 5xx and 429 maybe, 4xx no |

---

## Putting it together

A robust JSON GET combines the pieces: timeout, status check, and the `retry` helper from chapter 14, retrying only what's transient:

```clojure
(defn transient? [^js e]
  (let [{:keys [type status]} (ex-data e)]
    (or (= (.-name e) "TimeoutError")
        (= (ex-message e) "fetch failed")
        (and (= type :http-error) (or (= status 429) (>= status 500))))))

(defn ^:async get-json-robust [url]
  (await (retry (fn [] (get-json url)) {:attempts 3 :base-ms 200})))
```

Wire `transient?` into `retry`'s catch to rethrow non-transient errors immediately.

---

## Rules of thumb

- Check `.-ok` on every response. Non-2xx responses resolve.
- Read the body once, into a value.
- Give every request an `AbortSignal.timeout`.
- Find network error details in `.-cause`, and treat `TimeoutError` and `AbortError` as cancellation.
- Build URLs with `js/URL` and query strings with `searchParams`.
- Retry only transient failures (network errors, timeouts, 429, 5xx), and only idempotent requests.
- Hint `^js` on responses before reading their properties.
