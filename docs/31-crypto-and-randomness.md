# 31 — Crypto and randomness

*After this chapter you can tell which random sources are safe for which purpose, generate tokens and IDs, hash and sign data with `node:crypto`, compare secrets without timing leaks, and store passwords properly.*

---

## Two kinds of randomness

`cljs.core`'s random functions are built on `Math.random`:

```clojure
(rand)                 ;; a double in [0, 1)
(rand-int 10)          ;; 0–9
(rand-nth [:a :b])     ;; a random element
(shuffle (range 5))    ;; a random permutation
```

They're fine for simulations, jitter in retry backoff (chapter 14), sampling and tests. They are **not** fine for anything an attacker would benefit from predicting: tokens, session IDs, password reset links, API keys, nonces. `Math.random` isn't cryptographically secure, and its output can be predicted from earlier values.

> ⚠️ **Gotcha:** `random-uuid` is in the same category. In the ClojureScript 1.12.145 source, it builds the UUID from `rand-int`, which is `Math.random`. It's fine as a unique identifier and **unsafe as a secret**. A UUID in a password-reset URL must come from `crypto/randomUUID`.

For anything security-related, use `node:crypto`:

```clojure
(ns app.security
  (:require ["node:crypto" :as crypto]
            ["node:util" :as util]
            [clojure.string :as str]))

(.toString (crypto/randomBytes 32) "base64url")
;; => "Dk2OvPCAOlDAbVJULPfdNbfAwlgQPubxzTDHpNo32EY"   43 chars, 256 bits, URL-safe

(crypto/randomUUID)        ;; => "f154315e-e034-41dd-ab06-175aad61fae3"   secure v4 UUID
(uuid (crypto/randomUUID)) ;; as a ClojureScript UUID value

(crypto/randomInt 1 7)     ;; secure integer in [1, 7), e.g. a die roll
```

32 random bytes encoded as `base64url` is a good default token: unguessable, and safe in URLs and headers without escaping.

---

## Hashing

```clojure
(-> (crypto/createHash "sha256")
    (.update "hello")
    (.digest "hex"))
;; => "2cf24dba5fb0a30e26e83b2ac5b9e29e1b161e5c1fa7425e73043362938b9824"
```

`->` reads naturally here: create, feed data (call `.update` repeatedly for streaming input), then digest to `"hex"`, `"base64"` or a `Buffer`. Use SHA-256 for checksums, content addressing and cache keys. `(crypto/getHashes)` lists what's available.

A plain hash is **not** a way to store passwords or to authenticate messages. Those need the two tools below.

---

## HMAC: authenticating messages

An HMAC proves a message was produced by someone holding the secret key and hasn't been altered. That's what webhook signatures, signed cookies and API request signing use:

```clojure
(defn sign [secret msg]
  (-> (crypto/createHmac "sha256" secret)
      (.update msg)
      (.digest "hex")))

(sign "s3cret" "amount=100")
;; => "c9084003b5eedd11a5b4b0791cbcbeed0e017b175a3669d1e677c204b7242571"
```

### Constant-time comparison

To verify, compute the expected signature and compare. **Not with `=`**:

```clojure
(defn safe-equal? [a b]
  (let [ba (js/Buffer.from a)
        bb (js/Buffer.from b)]
    (and (= (.-length ba) (.-length bb))
         (crypto/timingSafeEqual ba bb))))

(safe-equal? (sign "s3cret" "amount=100") (sign "s3cret" "amount=100"))   ;; => true
(safe-equal? (sign "s3cret" "amount=100") (sign "s3cret" "amount=999"))   ;; => false
(safe-equal? "short" "longer")                                            ;; => false
```

`=` on strings stops at the first differing character, so how long it takes leaks how much of a guess was right. Over many requests an attacker can recover a valid signature one character at a time. `timingSafeEqual` always takes the same time.

> ⚠️ **Gotcha:** `timingSafeEqual` **throws** on buffers of different lengths, rather than returning `false`:
>
> ```clojure
> (crypto/timingSafeEqual (js/Buffer.from "a") (js/Buffer.from "ab"))
> ;; Throws: Input buffers must have the same byte length   (code ERR_CRYPTO_TIMING_SAFE_EQUAL_LENGTH)
> ```
>
> Check the lengths first, as `safe-equal?` does. A length mismatch leaks only the length, which for a fixed-size signature is public anyway.

---

## Passwords

Store passwords with a deliberately slow, salted key-derivation function. `node:crypto` has `scrypt` built in:

```clojure
(def scrypt (util/promisify crypto/scrypt))

(defn ^:async hash-password [password]
  (let [salt    (crypto/randomBytes 16)
        ^js key (await (scrypt password salt 64))]
    (str (.toString salt "hex") ":" (.toString key "hex"))))

(defn ^:async check-password [password stored]
  (let [[salt-hex key-hex] (str/split stored #":")
        ^js key (await (scrypt password (js/Buffer.from salt-hex "hex") 64))]
    (crypto/timingSafeEqual key (js/Buffer.from key-hex "hex"))))
```

```clojure
(let [stored (await (hash-password "correct horse"))]
  (await (check-password "correct horse" stored))   ;; => true
  (await (check-password "wrong" stored)))          ;; => false
```

- **A random salt per password** makes identical passwords hash differently and defeats precomputed tables. Store it alongside the hash.
- **Slow on purpose.** Each guess costs real CPU. The async version runs on Node's thread pool, so it doesn't block the event loop (chapter 14).
- **Constant-time comparison.** Both buffers are 64 bytes, so `timingSafeEqual` won't throw here.

Never store passwords with plain SHA-256, even salted. It's fast, and fast is the problem: a GPU tries billions of SHA-256 guesses per second.

---

## What not to do

- **Don't write your own crypto.** No custom ciphers, no clever hash constructions, no "encryption" with XOR. Use `node:crypto`'s primitives as documented.
- **Don't use `goog.crypt`** for security (chapter 29).
- **Don't put secrets in source or in `goog-define`.** Compile-time constants end up in the output file (chapter 21). Read secrets from the environment at runtime (chapter 26).
- **Don't log secrets.** Structured logging (chapter 27) makes it easy to log a whole request map, including its `authorization` header. Strip sensitive keys in the logger.

---

## Rules of thumb

- `rand`, `rand-int`, `shuffle` and `random-uuid` are for non-security uses only.
- Tokens: `(.toString (crypto/randomBytes 32) "base64url")`. Secret UUIDs: `crypto/randomUUID`.
- Use SHA-256 for checksums, HMAC-SHA-256 for authenticity, and scrypt for passwords.
- Compare secrets with `timingSafeEqual`, after checking lengths.
- Salt every password, and use the async scrypt so the event loop stays free.
- Keep secrets out of source, build output and logs.
