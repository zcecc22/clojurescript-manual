# 22 — Files and I/O

*After this chapter you can read, write and walk the filesystem with `node:fs/promises`, handle paths portably, work with `Buffer`s, and write files safely.*

---

## Setup

```clojure
(ns app.files
  (:require ["node:fs/promises" :as fsp]
            ["node:fs" :as fs]
            ["node:path" :as path]
            ["node:os" :as os]
            [clojure.string :as str]))
```

Use `node:fs/promises` for everything in application code. `node:fs` has the synchronous (`readFileSync`) and callback versions. The synchronous ones are fine in a CLI's startup code and wrong anywhere they'd block a server (chapter 14).

> ⚠️ **Gotcha:** A value that comes out of `await` isn't known to the compiler as a JavaScript object. Reading a property on it, like `(.-mtime st)` or `(.-syscall e)`, triggers an **inference warning**, and that warning means a broken release build (chapter 21). Very common names such as `.-code` and `.-name` happened not to warn, but don't rely on knowing which. Hint the bindings with `^js`: `(let [^js st (await (fsp/stat f))] ...)`, and `(catch :default ^js e ...)` for errors whose properties you read. All the examples below are hinted.

---

## Reading and writing whole files

```clojure
(await (fsp/writeFile f "line one\nline two\n"))
(await (fsp/readFile f "utf8"))      ;; => "line one\nline two\n"
(await (fsp/appendFile f "line three\n"))

(str/split-lines (await (fsp/readFile f "utf8")))
;; => ["line one" "line two" "line three"]
```

**Always pass an encoding** when you want text. Without one you get a `Buffer`:

```clojure
(type (await (fsp/readFile f)))   ;; => #object[Buffer]
```

Whole-file reads are right for config, templates and anything that comfortably fits in memory. For large files, stream them (chapter 13).

### Errors are `.-code`s

Filesystem errors are `Error`s with a string `code`, and that's what you branch on:

```clojure
(try
  (await (fsp/readFile "nope.txt" "utf8"))
  (catch :default ^js e
    [(.-code e) (.-syscall e)]))    ;; => ["ENOENT" "open"]
```

The common codes are `ENOENT` (doesn't exist), `EACCES` (permission denied), `EEXIST` (already exists), `ENOTDIR` and `EISDIR`.

> **Coming from imperative languages:** Don't check whether a file exists and then open it. Another process can delete it in between. Open it and handle `ENOENT`, the same advice as Go's `errors.Is(err, fs.ErrNotExist)` or Python's EAFP.

When you only need to know whether a path exists, `access` rejects if it doesn't:

```clojure
(defn ^:async path-exists? [p]
  (try (await (fsp/access p)) true
       (catch :default _ false)))
```

Don't call it `exists?`. That name is already taken by a `cljs.core` macro (chapter 09), and redefining it produces an `exists? already refers to: cljs.core/exists?` warning, which `:warnings-as-errors` turns into a failed build.

---

## Metadata and directories

```clojure
(let [^js st (await (fsp/stat f))]
  [(.-size st) (.isFile st) (.isDirectory st) (.-mtime st)])
;; => [29 true false #inst "..."]
```

Create nested directories in one call with `:recursive`. It doesn't fail if they already exist:

```clojure
(await (fsp/mkdir (path/join dir "a" "b") #js {:recursive true}))
```

Listing:

```clojure
(vec (await (fsp/readdir dir)))
;; => ["a" "notes.txt"]

(mapv (fn [^js d] [(.-name d) (.isDirectory d)])
      (await (fsp/readdir dir #js {:withFileTypes true})))
;; => [["a" true] ["notes.txt" false]]

(vec (sort (await (fsp/readdir dir #js {:recursive true}))))
;; => ["a" "a/b" "a/b/c.edn" "notes.txt"]
```

`:withFileTypes` saves a `stat` call per entry. `:recursive` (Node 20+) walks the whole tree and returns relative paths.

Remove a tree, without failing if it's already gone:

```clojure
(await (fsp/rm dir #js {:recursive true :force true}))
```

Temporary directories:

```clojure
(await (fsp/mkdtemp (path/join (os/tmpdir) "manual-")))
;; => "/tmp/manual-a1B2c3"
```

---

## Paths

Build paths with `node:path`, never with string concatenation:

```clojure
(path/join "a" "../b" "c")           ;; => "b/c"       joins and normalises
(path/resolve "rel")                 ;; => absolute, against the current directory
(path/basename "/x/y.txt" ".txt")    ;; => "y"
(path/dirname "/x/y.txt")            ;; => "/x"
(path/extname "a.tar.gz")            ;; => ".gz"
(js->clj (path/parse "/home/u/f.txt") :keywordize-keys true)
;; => {:root "/", :dir "/home/u", :base "f.txt", :ext ".txt", :name "f"}
```

> ⚠️ **Gotcha:** `js/__dirname` in a shadow-cljs build is the directory of the **output** file (`out/`), not of your source. Relative paths resolve against `(js/process.cwd)`, the directory the user ran the command from. A CLI that reads files bundled with it should resolve them from the script's location, and one that reads the user's files should resolve from the current directory. Decide which you mean every time.

> ⚠️ **Gotcha:** Never join untrusted input onto a base directory and assume the result stays inside it. `(path/join base "../../etc/passwd")` escapes. Resolve the result and check that it starts with the resolved base plus a separator.

---

## Buffers

A `Buffer` is Node's byte array. Encoding conversions go through it:

```clojure
(js/Buffer.from "héllo")                     ;; => #object[Buffer héllo]
(.-length (js/Buffer.from "héllo"))          ;; => 6    bytes
(count "héllo")                              ;; => 5    UTF-16 units
(.toString (js/Buffer.from "héllo") "hex")   ;; => "68c3a96c6c6f"
(.toString (js/Buffer.from "aGk=" "base64")) ;; => "hi"
```

Use `.-length` for byte sizes, such as `Content-Length`, and `count` only for string lengths.

### Low-level reads

For random access, open a file handle and read into a buffer. Close it in `finally`:

```clojure
(let [fh (await (fsp/open f "r"))]
  (try
    (let [buf    (js/Buffer.alloc 3)
          ^js r  (await (.read fh buf 0 3 0))]   ; buffer, offset, length, position
      [(.-bytesRead r) (.toString buf "utf8")])
    (finally
      (await (.close fh)))))
```

Leaking file handles is the classic Node resource leak. Every `open` gets a `finally`.

---

## Writing safely

A crash or a full disk halfway through `writeFile` leaves a truncated file. For anything that matters, like config, state or a data file, write to a temporary file in the **same directory** and rename it over the target. A rename within one filesystem is atomic, so readers see either the old file or the new one, never half of either:

```clojure
(defn ^:async write-atomic! [f content]
  (let [tmp (str f ".tmp")]
    (await (fsp/writeFile tmp content))
    (await (fsp/rename tmp f))))
```

For EDN data, which round-trips ClojureScript values, write `(pr-str data)` and read it back with `clojure.edn/read-string` (chapter 23).

---

## Rules of thumb

- Use `node:fs/promises` in application code. Sync APIs only at CLI startup.
- Always pass `"utf8"` when you want a string.
- Handle `ENOENT` instead of checking for existence first.
- Hint awaited Node objects with `^js` before reading their properties.
- Build paths with `path/join` and `path/resolve`, and guard untrusted path segments.
- Close every file handle in `finally`.
- Write important files atomically: temp file in the same directory, then rename.
