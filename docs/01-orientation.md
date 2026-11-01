# 01 — Orientation

*After this chapter you can set up a shadow-cljs project, build and run it on Node, and work in a REPL connected to running code.*

---

## The model

ClojureScript is Clojure, a Lisp built around immutable data, compiled to JavaScript. The pipeline has three stages:

1. **The ClojureScript compiler** turns `.cljs` files into JavaScript. It runs on the JVM, which is why you need a JDK even though nothing you ship runs on Java.
2. **The Google Closure Compiler** optimises that JavaScript. In development it does almost nothing. In a release build it renames, inlines and removes dead code across your whole program, including `cljs.core` itself.
3. **Node** runs the result.

**shadow-cljs** drives all three. It is the build tool: it reads `shadow-cljs.edn`, fetches Clojure dependencies, compiles, watches files, hot-reloads, and hosts the REPL. It also understands `node_modules`, so requiring a Node built-in or an npm package is a one-liner.

The language is small. Almost everything is a function call written as a list: `(f a b)` calls `f` with `a` and `b`. Data is immutable by default. Functions take data and return new data. Mutation exists, but you ask for it explicitly (chapter 11).

> **Coming from imperative languages:** The thing to unlearn first is the statement. There are no statements. `if` returns a value, `let` returns a value, and a function returns whatever its last expression evaluates to. Once that clicks, most of the syntax follows.

---

## A project from scratch

You need Node, npm and a JDK. shadow-cljs is installed per project, not globally:

```sh
mkdir hello && cd hello
echo '{"name":"hello","private":true}' > package.json
npm i -D shadow-cljs
mkdir -p src/hello
```

`shadow-cljs.edn` describes the builds. It's written in EDN, Clojure's data notation:

```clojure
{:source-paths ["src"]
 :dependencies []
 :builds
 {:app {:target    :node-script
        :main      hello.core/main
        :output-to "out/hello.js"}}}
```

- `:source-paths` is where your code lives.
- `:dependencies` lists Clojure libraries from Maven/Clojars. This manual keeps it empty.
- `:target :node-script` produces a single runnable Node script.
- `:main` is the function called at startup, with the command-line arguments.

The namespace `hello.core` lives at `src/hello/core.cljs`. The file path must match the namespace name:

```clojure
(ns hello.core)

(defn greet [name]
  (str "Hello, " name "!"))

(defn main [& args]
  (println (greet (or (first args) "world")))
  (prn args))
```

Build it and run it:

```sh
npx shadow-cljs compile app
node out/hello.js
```

```
Hello, world!
nil
```

```sh
node out/hello.js Ada Lovelace
```

```
Hello, Ada!
("Ada" "Lovelace")
```

`main` receives the arguments after the script name as strings. With none it receives `nil`, not an empty list. `(first nil)` is `nil`, so `or` falls back to `"world"`.

> ⚠️ **Gotcha:** Namespace segments with hyphens map to underscores on disk. `my-app.core` must live at `src/my_app/core.cljs`. Get it wrong and the build fails with `The required namespace "my-app.core" is not available.`, with no hint about the underscore.

---

## The three commands you'll use

| Command | What it does |
|---|---|
| `npx shadow-cljs compile app` | One development build. Fast, readable output, not self-contained |
| `npx shadow-cljs watch app` | Development build that recompiles and hot-reloads on every save |
| `npx shadow-cljs release app` | Optimised, self-contained build for shipping |

The development output is a small loader that pulls compiled files from `.shadow-cljs/`, so copying `out/hello.js` somewhere else won't work. Only `release` output is a standalone file you can ship.

Each command starts a JVM, which takes several seconds. Run `npx shadow-cljs server` in a spare terminal, and later commands connect to that server instead of starting their own JVM.

---

## The REPL

Clojure development centres on a REPL connected to the running program. You change a function, evaluate it, and the running process picks up the new definition without a restart.

The quickest one needs no build:

```sh
npx shadow-cljs node-repl
```

```
cljs.user=> (+ 1 2)
3
cljs.user=> (require '[clojure.string :as s])
nil
cljs.user=> (s/upper-case "hi")
"HI"
cljs.user=> js/process.version
"v20.19.2"
cljs.user=> (require '[hello.core :as h])
nil
cljs.user=> (h/greet "REPL")
"Hello, REPL!"
```

This starts a Node process and attaches to it. Everything on your `:source-paths` is requirable.

The real workflow is editor-integrated: shadow-cljs starts an nREPL server (the port is written to `.shadow-cljs/nrepl.port`), and your editor (CIDER, Calva, Conjure, Cursive) connects to it. You then evaluate forms straight from the source file. Learning that keybinding is the single biggest productivity step in this language.

> **Coming from imperative languages:** In Go you'd edit, rebuild, rerun. In Python you might try snippets in a REPL and paste them into the file. Here the file *is* the REPL input: you put the cursor on a `defn`, evaluate it, and test it from a comment block below. The program stays running the whole time.

---

## What ClojureScript deliberately lacks

Things you'll reach for and not find:

- **Classes and inheritance.** You model data with maps and get polymorphism from protocols and multimethods (chapters 15 and 16). `deftype` exists for the rare case where you need a JS class.
- **Mutable variables.** `def` and `let` bind names to values, and those bindings don't change. When you need mutable state you use an atom (chapter 11).
- **Statements and `return`.** Everything is an expression (chapter 05).
- **Loops with counters.** `for` is a sequence comprehension, not a loop. You use `map`, `filter` and `reduce`, or `loop`/`recur` when you really need iteration (chapters 05 and 07).
- **Threads.** It's JavaScript: one thread, one event loop. There are no locks because there's nothing to lock against (chapter 14).
- **Integers.** Every number is a JavaScript double (chapter 03).
- **Runtime `eval`.** Code is compiled ahead of time on the JVM, so the running program has no compiler to call. `eval` exists in `cljs.core`, but calling it throws `cljs.core/*eval* not bound`.

---

## The editor matters

Lisp syntax is unpleasant without structural editing and pleasant with it. Before writing much code, set up:

- **Paren matching and rainbow parens**, so you can see the structure.
- **Structural editing** (paredit or parinfer), so parens stay balanced and you edit forms rather than characters.
- **REPL integration**: evaluate the form at the cursor, evaluate the top-level form, load the file.

Calva for VS Code works with shadow-cljs out of the box. So do CIDER for Emacs and Conjure for Neovim.

---

## Rules of thumb

- One namespace per file, and the path must match the namespace, with `-` becoming `_`.
- Keep `npx shadow-cljs server` running while you work. Every command after the first is then fast.
- Develop against `watch` and a connected REPL. Use `compile` only for one-off checks.
- Ship only `release` output. Development builds aren't self-contained.
- If you're typing a whole file and then running it, you're not using the REPL yet. Evaluate forms as you write them.
