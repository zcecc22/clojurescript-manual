# The ClojureScript Manual

A working reference for ClojureScript on Node, written for people who already program well in an imperative language.

**Targets ClojureScript 1.12.145, built with shadow-cljs 3.5.8, running on Node 20.19.2** (JDK 21 for the compiler). Anything that needs a specific version is marked inline in the source, e.g. `<!-- CLJS 1.12+ -->`. Every version-gated claim was checked against that real install rather than recalled.

**Node, compiled with shadow-cljs. Nothing else.** No browser, no nbb, no self-hosted ClojureScript. The output of every example is a Node script.

**Core only.** What ships with ClojureScript (`cljs.core`, `clojure.string`, `clojure.set`, `clojure.walk`, `clojure.edn`, `cljs.reader`, `cljs.pprint`, `cljs.test`, `cljs.spec.alpha`), the Google Closure Library that comes bundled with it, and Node's built-in modules. No core.async, no promesa, no npm packages. Where the ecosystem has an obvious answer that core lacks, the manual shows you how to hand-roll it. Knowing the shape of the thing is what makes the library version readable later.

---

## Who this is for

You already know Go, JavaScript/TypeScript, Python, Java, or C#. You do not need to be told what a function is. What you need is:

- the places where a functional Lisp is genuinely different from the imperative model you're used to,
- what idiomatic ClojureScript actually looks like, as opposed to JavaScript written in parentheses,
- the specific traps that come from running on a JavaScript runtime.

So the chapters skip the fundamentals and spend their budget on those three things.

---

## Reading order

Chapters are grouped in **tiers by priority**, not by difficulty. Read Tier 1 in order, because it builds. After that the tiers are largely independent, so take them when you need them.

The tier list below is the authority on reading order. File numbers are just stable identifiers.

### Tier 1 — Core language

Everything else assumes these. Read all ten, in order.

| # | Chapter | Covers |
|---|---|---|
| 01 | [Orientation](docs/01-orientation.md) | The CLJS model, a shadow-cljs project, the REPL workflow, what CLJS lacks |
| 02 | [Syntax and evaluation](docs/02-syntax-and-evaluation.md) | Forms, literals, symbols vs keywords, truthiness, `=` vs `identical?` |
| 03 | [Values and types](docs/03-values-and-types.md) | Numbers are doubles, strings and "chars", `nil` vs `undefined`, keywords |
| 04 | [Functions](docs/04-functions.md) | `defn`, multi-arity, variadics, destructuring, `#()`, `partial`/`comp`, closures |
| 05 | [Control flow](docs/05-control-flow.md) | `if`/`when`/`cond`/`case`, `let`, `loop`/`recur`, life without early return |
| 06 | [Collections](docs/06-collections.md) | Persistent vectors, maps, sets, lists; `assoc`/`update`/`get-in`; equality |
| 07 | [Sequences](docs/07-sequences.md) | The seq abstraction, `map`/`filter`/`reduce`, `into`, laziness traps |
| 08 | [Namespaces](docs/08-namespaces.md) | The `ns` form, `:require`, requiring Node modules, `.cljc` |
| 09 | [JS interop](docs/09-js-interop.md) | `.method`, `.-prop`, `js/`, `#js`, `clj->js`/`js->clj`, renaming under advanced |
| 10 | [Errors](docs/10-errors.md) | `ex-info`/`ex-data`, `catch :default`, thrown non-Errors |

### Tier 2 — State and async

Read 11–13 in order; 14 is a pattern catalogue to return to.

| # | Chapter | Covers |
|---|---|---|
| 11 | [State](docs/11-state.md) | Atoms, `swap!`, watches, validators, volatiles |
| 12 | [Promises and async](docs/12-promises-and-async.md) | `js/Promise` interop, `^:async`/`await`, error propagation |
| 13 | [Events and streams](docs/13-events-and-streams.md) | `EventEmitter`, Node streams, async iteration |
| 14 | [Async patterns](docs/14-async-patterns.md) | The event loop, timeouts, `AbortController`, concurrency limits, workers, shutdown |

### Tier 3 — Abstraction and engineering

| # | Chapter | Covers |
|---|---|---|
| 15 | [Protocols and records](docs/15-protocols-and-records.md) | `defprotocol`, `defrecord`, `deftype`, `reify`, extending JS types |
| 16 | [Multimethods](docs/16-multimethods.md) | `defmulti`/`defmethod`, dispatch functions, hierarchies |
| 17 | [Macros](docs/17-macros.md) | Macro namespaces, syntax-quote, hygiene, and when *not* to |
| 18 | [Transducers](docs/18-transducers.md) | Composable transformations without intermediate sequences |
| 19 | [Spec](docs/19-spec.md) | `cljs.spec.alpha` for validation, conformance, instrumentation |
| 20 | [Testing](docs/20-testing.md) | `cljs.test`, the `:node-test` target, async tests, fixtures |
| 21 | [Tooling](docs/21-tooling.md) | shadow-cljs builds, `release`, advanced compilation, externs, source maps |

### Tier 4 — Node for real work

Take these as your projects demand them.

| # | Chapter | Covers |
|---|---|---|
| 22 | [Files and I/O](docs/22-files-and-io.md) | `node:fs/promises`, `node:path`, streaming |
| 23 | [Data formats](docs/23-data-formats.md) | JSON, EDN, `cljs.pprint` |
| 24 | [HTTP servers](docs/24-http-servers.md) | `node:http`, routing by hand, middleware as composition, shutdown |
| 25 | [HTTP clients](docs/25-http-clients.md) | `js/fetch`, timeouts, retries |
| 26 | [CLI tools](docs/26-cli-tools.md) | Arguments, `parseArgs`, exit codes, `child_process`, signals, env |
| 27 | [Time and logging](docs/27-time-and-logging.md) | `js/Date` pitfalls, `goog.date`, structured logging |

### Tier 5 — Advanced and occasional

| # | Chapter | Covers |
|---|---|---|
| 28 | [Performance](docs/28-performance.md) | Transients, JS arrays and objects, type hints, reading compiled output |
| 29 | [Closure Library](docs/29-closure-library.md) | `goog.object`, `goog.string`, `goog.functions` and friends |
| 30 | [Shipping code](docs/30-shipping-code.md) | `:node-library` and `:esm` targets, calling CLJS from JS |
| 31 | [Crypto and randomness](docs/31-crypto-and-randomness.md) | `node:crypto`, `rand` vs a CSPRNG, HMAC, constant-time comparison |

### Reference

- **[STYLE.md](STYLE.md)** — the style guide, as decidable rules with examples. Read it after Tier 1; reread it after Tier 3.
- **[CHEATSHEET.md](CHEATSHEET.md)** — one page of syntax and idioms, for when you know what you want and forget how to spell it.

---

## Conventions used throughout

Callouts are consistent, so you can skim for them.

> **Coming from imperative languages:** Where ClojureScript's behaviour will surprise you because of habits from Go, JavaScript, Python, Java and the like: mutation, loops, classes, statements, early returns.

> ⚠️ **Gotcha:** A trap that bites real people in real code. These are the highest-value paragraphs in the manual.

Every chapter opens with one line on what you'll be able to do afterwards, and closes with **Rules of thumb**, short statements you can actually apply while writing code.

Code that intentionally fails is labelled **`;; Does not compile`** or **`;; Throws`**. Everything else has been compiled with shadow-cljs and run on Node, so the output shown is what was observed, not predicted. Results are shown as `;; =>` comments, and printed output as `;; prints:`.

---

## The five-minute version

If you read nothing else, these are the differences that cause the most early pain:

1. **Data is immutable.** `assoc`, `conj` and `update` return a *new* collection and leave the old one untouched. Nothing you hold will change under you. → [Ch. 06](docs/06-collections.md)
2. **Everything is an expression.** `if`, `let` and `do` all return values. There are no statements and no `return`: a function returns its last expression. → [Ch. 05](docs/05-control-flow.md)
3. **Only `nil` and `false` are falsy.** `0`, `""` and empty collections are all truthy, unlike in JavaScript and Python. → [Ch. 02](docs/02-syntax-and-evaluation.md)
4. **Numbers are JavaScript doubles.** There are no integers, no ratios, and no overflow errors, just `0.30000000000000004` and silent precision loss above 2⁵³. → [Ch. 03](docs/03-values-and-types.md)
5. **Plain maps replace classes.** You model a domain with maps and keywords, write functions over them, and reach for protocols and records only when you need polymorphism. → [Ch. 06](docs/06-collections.md), [Ch. 15](docs/15-protocols-and-records.md)
