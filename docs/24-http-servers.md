# 24 — HTTP servers

*After this chapter you can build an HTTP service on `node:http` where handlers are plain functions from request maps to response maps, compose middleware as function wrappers, and shut down without dropping requests.*

---

## The design: requests and responses as data

Node's `http` API hands you mutable `req` and `res` objects. The idiomatic ClojureScript approach is to convert at the edge:

1. Turn the Node request into a **map**: `{:method :get, :path "/users/1", :headers {...}, :body "..."}`.
2. Call a **handler**, a plain function from request map to response map.
3. Write the response map, `{:status 200, :headers {...}, :body "..."}`, back to Node.

This is the "Ring" model from JVM Clojure. The payoff is that handlers and middleware are pure functions over data. You test them by calling them with maps, with no server and no mocks.

```clojure
(ns app.server
  (:require ["node:http" :as http]
            [clojure.string :as str]))
```

---

## The edge: Node to maps and back

Reading the body means consuming the request stream. `req` is an async iterable of `Buffer` chunks (chapter 13):

```clojure
(defn ^:async read-body [^js req]
  (let [chunks #js []
        it     (.call (aget req js/Symbol.asyncIterator) req)]
    (loop []
      (let [^js r (await (.next it))]
        (when-not (.-done r)
          (.push chunks (.-value r))
          (recur))))
    (.toString (js/Buffer.concat chunks) "utf8")))
```

Concatenating the buffers *before* decoding matters: decoding chunk by chunk can split a multi-byte character.

```clojure
(defn ^:async ->request [^js req]
  (let [url (js/URL. (.-url req) "http://localhost")]
    {:method  (keyword (str/lower-case (.-method req)))
     :path    (.-pathname url)
     :query   (into {} (map vec) (js/Array.from (.entries (.-searchParams url))))
     :headers (js->clj (.-headers req))       ; lower-cased string keys
     :body    (await (read-body req))}))

(defn send! [^js res {:keys [status headers body]}]
  (.writeHead res (or status 200) (clj->js (or headers {})))
  (.end res body))
```

`req.url` is only the path and query, so `js/URL` needs a base to parse it. Headers stay as string keys because they're user-controlled (chapter 23).

> ⚠️ **Gotcha:** `read-body` buffers the whole body with no limit, so one client can send a gigabyte and exhaust memory. In production, count bytes as you push chunks and reject the request with 413 past a limit.

---

## Handlers and routing

A helper for JSON responses, and a router that's just a function:

```clojure
(defn json [status data]
  {:status  status
   :headers {"content-type" "application/json"}
   :body    (js/JSON.stringify (clj->js data))})

(def users (atom {1 {:id 1 :name "Ada"}}))

(defn router [{:keys [method path body]}]
  (let [segments (vec (rest (str/split path #"/")))]
    (case [method (first segments)]
      [:get "health"] (json 200 {:ok true})
      [:get "users"]  (if-let [id (some-> (second segments) parse-long)]
                        (if-let [u (get @users id)]
                          (json 200 u)
                          (json 404 {:error "not found"}))
                        (json 200 (vals @users)))
      [:post "users"] (let [u  (js->clj (js/JSON.parse body) :keywordize-keys true)
                            id (inc (apply max (keys @users)))]
                        (swap! users assoc id (assoc u :id id))
                        (json 201 (get @users id)))
      (json 404 {:error "no route" :path path}))))
```

`case` on a vector of `[method first-segment]` is a compact router for a small service. It dispatches on literal constants (chapter 05), and the default clause is the 404. A larger service can use a map of routes, or a multimethod dispatching on `[method route-name]` (chapter 16).

---

## Middleware is function composition

Middleware takes a handler and returns a new handler. Cross-cutting concerns wrap the router:

```clojure
(defn wrap-errors [handler]
  (fn [req]
    (try
      (handler req)
      (catch :default e
        (prn :handler-error (ex-message e) (ex-data e))
        (json 500 {:error "internal error"})))))

(defn wrap-logging [handler]
  (fn [req]
    (let [res (handler req)]
      (println (name (:method req)) (:path req) "->" (:status res))
      res)))

(def app (-> router wrap-errors wrap-logging))
```

With `->`, the last wrapper is the outermost: a request passes through logging, then error handling, then the router. Authentication, request IDs, CORS headers and body parsing are all middleware of the same shape.

> **Coming from imperative languages:** This is the same pattern as Go's `func(http.Handler) http.Handler` middleware and Express's `app.use`. The difference is that the request is an immutable map, so middleware adds information with `assoc` before passing it down (`(handler (assoc req :user user))`) rather than mutating a shared object.

---

## Starting the server

```clojure
(defn ^:async handle [req res]
  (send! res (app (await (->request req)))))

(defn start! [port]
  (let [server (http/createServer
                 (fn [req res]
                   (.catch (handle req res)
                           (fn [e] (prn :fatal e) (.end res)))))]
    (js/Promise. (fn [resolve _] (.listen server port #(resolve server))))))
```

The `.catch` matters: `handle` is async, and an unhandled rejection would crash the server (chapter 12). Exercised with `fetch` (chapter 25):

```
get /health -> 200          [200 {:ok true}]
get /users/1 -> 200         [200 {:id 1, :name "Ada"}]
get /users/99 -> 404        [404 {:error "not found"}]
post /users -> 201          [201 {:name "Bob", :id 2}]
get /boom -> 500            [500 {:error "internal error"}]
get /nope -> 404            [404 {:error "no route", :path "/nope"}]
```

Listening on port `0` picks a free port, which you read back with `(.-port (.address server))`. That's how tests run a real server without port collisions.

### Async handlers

The router above is synchronous. When handlers need I/O, make them return promises, and make the middleware and `handle` `await` them. Every layer has to agree on whether it's sync or async, so it's simplest to pick async for the whole chain from the start.

---

## Graceful shutdown

`.close` stops accepting new connections and calls back once in-flight requests finish:

```clojure
(.close server (fn [] (prn :closed)))
```

Verified with a request that takes 200 ms, in flight when `close` was called:

```
:new-request [:refused "ECONNREFUSED"]     a new connection after close
:in-flight "slow done"                     the existing request completed
:closed
```

Combined with signal handling (chapter 14):

```clojure
(defn install-shutdown! [^js server]
  (doseq [sig ["SIGINT" "SIGTERM"]]
    (.once js/process sig
           (fn []
             (.close server #(js/process.exit 0))
             (.unref (js/setTimeout #(js/process.exit 1) 10000))))))
```

The timer is the deadline. If connections don't drain in ten seconds, the process exits anyway, and `.unref` stops the timer from keeping a cleanly-closed process alive.

---

## Things to configure before production

- **Timeouts.** `server.requestTimeout` and `server.headersTimeout` bound slow clients, and `server.keepAliveTimeout` controls idle connections. Set them explicitly rather than relying on defaults.
- **Body limits**, as noted above.
- **A reverse proxy** for TLS termination, compression and static files is conventional. `node:https` exists if you need TLS in-process.

---

## Rules of thumb

- Convert at the edge: Node `req` to map, response map to Node `res`.
- Handlers are functions from request map to response map. Test them without a server.
- Write middleware as `(fn [handler] (fn [req] ...))` and compose with `->`.
- Catch errors in middleware (500) and at the top-level promise (never crash).
- Bound request bodies and set server timeouts.
- Listen on port 0 in tests.
- On `SIGTERM`: `.close`, wait for in-flight requests, exit, with a deadline.
