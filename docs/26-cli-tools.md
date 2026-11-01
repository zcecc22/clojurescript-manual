# 26 — CLI tools

*After this chapter you can build a command-line tool that parses flags with `parseArgs`, reads stdin, writes to stdout and stderr, exits with meaningful codes without losing output, reads configuration from the environment, and runs other programs.*

---

## The shape of a CLI

A `:node-script` build is already an executable script. shadow-cljs writes the `#!/usr/bin/env node` line itself:

```sh
npx shadow-cljs release tool
chmod +x out/tool.js
./out/tool.js args -v file1
```

> ⚠️ **Gotcha:** Don't add a shebang yourself with `:prepend`. The `:node-script` output already starts with one, and `:prepend` inserts a second copy *inside* the generated wrapper, where it's a syntax error: `SyntaxError: Invalid or unexpected token`.

The skeleton: `main` receives the arguments as strings, calls an async `run`, and turns any failure into a message and an exit code:

```clojure
(ns tool.core
  (:require ["node:util" :as util]
            ["node:child_process" :as cp]
            [clojure.string :as str]))

(defn main [& args]
  (-> (run args)
      (.catch (fn [e]
                (.write js/process.stderr (str "fatal: " (ex-message e) "\n"))
                (set! (.-exitCode js/process) 1)))))
```

---

## Parsing flags with `parseArgs`

`node:util` has a flag parser built in:

```clojure
(def options
  (clj->js {:verbose {:type "boolean" :short "v"}
            :output  {:type "string"  :short "o" :default "out.txt"}
            :tag     {:type "string"  :multiple true}}))

(defn parse [args]
  (let [^js r (util/parseArgs #js {:args (clj->js args)
                                   :options options
                                   :allowPositionals true})]
    {:opts        (js->clj (js/Object.assign #js {} (.-values r)) :keywordize-keys true)
     :positionals (vec (.-positionals r))}))
```

```sh
./out/tool.js args -v --tag a --tag b file1 -o x.txt
# {:verbose true, :tag ["a" "b"], :output "x.txt"} ["args" "file1"]

./out/tool.js args
# {:output "out.txt"} ["args"]
```

Unknown flags throw, with a message good enough to show the user directly:

```
fatal: Unknown option '--bogus'. To specify a positional argument starting with a '-', ...
```

Two traps showed up writing this example, both worth remembering:

> ⚠️ **Gotcha:** The options object is nested, and `#js` is shallow (chapter 09). Written as `#js {:verbose {:type "boolean"}}`, the inner maps stay ClojureScript maps and `parseArgs` fails with `The "options.verbose.type" property must be ('string|boolean'). Received undefined`. Use `clj->js` for nested options.

> ⚠️ **Gotcha:** `parseArgs` returns `.values` as an object with a **null prototype**, and `js->clj` doesn't convert those. It returned `#object[Object]` unchanged. Copy it into a plain object first with `(js/Object.assign #js {} values)`. Other Node APIs return null-prototype objects too, such as `querystring.parse`, so if `js->clj` hands back a JS object, this is why.

For subcommands, take the first positional and dispatch with `case`, or with a multimethod if the set of commands is open (chapter 16).

---

## stdin, stdout, stderr

Write output with `.write` on the streams. It doesn't append a newline, and it lets you separate results (stdout) from diagnostics (stderr):

```clojure
(.write js/process.stdout result)
(.write js/process.stderr "error: something broke\n")
```

`println` writes to stdout too, but through `console.log` (chapter 05), so prefer explicit writes for output other programs will parse.

Reading all of stdin when input is piped in uses the async-iteration helper from chapter 13:

```clojure
(defn ^:async read-stdin []
  (let [chunks #js []
        it     (.call (aget js/process.stdin js/Symbol.asyncIterator) js/process.stdin)]
    (loop []
      (let [^js r (await (.next it))]
        (when-not (.-done r)
          (.push chunks (.-value r))
          (recur))))
    (.toString (js/Buffer.concat chunks) "utf8")))
```

```sh
echo "hello stdin" | ./out/tool.js upper
# HELLO STDIN
```

`(.-isTTY js/process.stdout)` is `true` for a terminal and `nil` when output is piped. Use it to decide whether to print colours or progress bars.

---

## Exit codes, without losing output

Exit codes are how scripts and CI know what happened: 0 for success, non-zero for failure. Set `process.exitCode` and let the process end naturally:

```clojure
(.write js/process.stderr "error: something broke\n")
(set! (.-exitCode js/process) 2)
```

```sh
./out/tool.js fail; echo "exit=$?"
# error: something broke
# exit=2
```

> ⚠️ **Gotcha:** Don't call `(js/process.exit n)` after writing output. When stdout is a pipe, writes are asynchronous, and `process.exit` kills the process before they flush. Writing 5,000,000 bytes and then calling `process.exit(0)` delivered **65,536** bytes to `wc -c`. Setting `exitCode` instead delivered all 5,000,000. Keep `process.exit` for emergencies, like a shutdown deadline (chapter 14).

Conventions worth following: `1` for general failure, `2` for misuse (bad arguments), and `64`–`78` from BSD `sysexits.h` if you want finer distinctions (`64` is "usage error").

---

## Configuration from the environment

```clojure
(or (.. js/process -env -APP_PORT) "8080")
```

```sh
APP_PORT=9000 ./out/tool.js env    # "9000"
./out/tool.js env                  # "8080"
```

Environment variables are always strings, or `nil` when unset. Parse and validate them once at startup into a config map, with `parse-long` and a spec (chapter 19), and fail fast with a clear message rather than discovering a bad `PORT` on the first request.

Precedence is conventionally: flags over environment over config file over defaults. With maps that's a `merge`, defaults first and flags last.

---

## Running other programs

`execFile` runs a program with an argument array, without a shell, and collects its output. Promisify it once:

```clojure
(def exec-file (util/promisify cp/execFile))

(let [^js r (await (exec-file "ls" #js ["-1" "src"]))]
  (str/split-lines (str/trim (.-stdout r))))
;; => ["app" "scratch" "tool"]
```

A non-zero exit rejects, and the error carries the code and output:

```clojure
(try
  (await (exec-file "ls" #js ["/nope"]))
  (catch :default ^js e
    [(.-code e) (str/trim (.-stderr e))]))
;; => [2 "ls: cannot access '/nope': No such file or directory"]
```

> ⚠️ **Gotcha:** Prefer `execFile` to `exec`. `exec` runs its string through a shell, so a file name containing spaces, quotes or `; rm -rf ~` becomes part of a command. `execFile` passes arguments directly, with no shell parsing.

`execFile` buffers output in memory, up to a `maxBuffer` limit of 1 MiB by default. For large output or long-running processes, use `cp/spawn` and treat `.-stdout` as a stream (chapter 13). `{:stdio "inherit"}` connects the child straight to your terminal.

---

## Signals

A long-running CLI should clean up on Ctrl-C:

```clojure
(.once js/process "SIGINT"
       (fn []
         (.write js/process.stderr "\ninterrupted\n")
         (set! (.-exitCode js/process) 130)    ; 128 + SIGINT(2), the shell convention
         (cleanup!)))
```

Registering a `SIGINT` handler replaces Node's default of exiting immediately, so your handler must make sure the process actually ends, by closing whatever keeps the event loop alive.

---

## Rules of thumb

- Ship the `release` build. It's one file with its own shebang.
- Parse flags with `util/parseArgs`, building options with `clj->js` and converting `.values` with `Object.assign` first.
- Results go to stdout, diagnostics to stderr.
- Set `process.exitCode`. Never `process.exit` after writing output.
- Read the environment once at startup into a validated config map.
- Run programs with `execFile` or `spawn` and an argument array, never a shell string.
