---
name: nrepl-eval
description: >-
  Evaluate Clojure code and run tests inside a Clojure project's already-running
  nREPL (its dev REPL) from the shell, using the `nreplctl` CLI. Use this
  whenever you work in a Clojure project (deps.edn / project.clj, a `.nrepl-port`
  file, or the user mentions a running repl/nrepl) and need to run or re-run
  tests (clojure.test, kaocha, eftest), reload changed namespaces, check what a
  function returns, or inspect the live system. Strongly prefer it over cold
  `clojure -M:test`, `lein test`, or `bb test`, which boot a fresh JVM every time
  — the running nREPL is warm and holds the project's loaded state (the integrant
  system, DB pool, caches), so it is far faster and can test against the live
  system. Triggers: "run the tests", "run this test namespace / var", "eval this
  in the repl", "does this work in the running repl", "is the test passing",
  "reload and test", or any request to exercise Clojure code against a live REPL.
---

# Evaluating Clojure in a project's running nREPL with nreplctl

A Clojure project usually has a long-lived **dev nREPL** running (started with
`clj -M:dev:nrepl`, an editor jack-in, or similar). It is a warm JVM with the
whole project loaded: its namespaces, its integrant system, its database pool.
`clojure -M:test` and friends boot a fresh JVM and reload everything on every
run, which costs 10-30s and discards all state. Evaluating in the live nREPL is
near-instant and can exercise the running system. `nreplctl` turns that into a
one-shot shell command you can drive.

## The tool

```
nreplctl eval '<code>' [-p/--port N] [--port-file PATH] [--host HOST] [-t/--timeout SECONDS]
```

- **Port**: `--port` wins; otherwise the port in `--port-file` (default
  `.nrepl-port` at the project root). Run from the project root and it just works.
- **Output**: stdout carries evaluated values and `out`; stderr carries `err`
  and nreplctl's diagnostics. Every value prints on its own line.
- **Exit codes**: `0` the eval ran, `1` the code threw or timed out, `2`
  couldn't resolve the port / connect / talk nREPL.

Each `nreplctl` call is a fresh nREPL session, but the runtime is shared: a
`def`, `require`, or reload in one call is visible to the next (namespaces are
global to the JVM). Only the *current namespace* resets — every call starts in
`user`.

## 1. Orient in the project

Two things to find before you do anything:

**The nREPL.** Sanity-check it: `nreplctl eval '(+ 1 1)'` should print `2`.
If you get exit `2` (`Could not connect …`), the `.nrepl-port` file is often
**stale** — it names a port whose server has exited. Don't assume the REPL is
down. Find the live one and pass `--port`:

```bash
ss -ltnp 2>/dev/null | grep -iE 'java|clj' \
  || lsof -nP -iTCP -sTCP:LISTEN 2>/dev/null | grep -iE 'java|clojure'
```

**The project's REPL workflow.** This is the key move for a Clojure project.
Read `dev/user.clj` (or `dev/dev.clj`) and the `:dev`/`:test` aliases in
`deps.edn`/`project.clj`. Most projects expose helpers in the `user`/`dev`
namespace that encode their reload and test conventions — use them instead of
reconstructing the setup:

- a **reset/reload**: `(user/reset)` (integrant restart), `(reload/reload)`
  (clj-reload), `((requiring-resolve 'clojure.tools.namespace.repl/refresh))`.
- a **test runner**: `(user/run-tests)`, `(user/run-tests 'ns/var)`, often
  wrapping kaocha or eftest and reloading first.

Example — a real project's `dev/user.clj` exposing exactly this:

```clojure
(defn reset [] (ig-repl/reset))                       ; restart the integrant system
(defn run-tests
  ([] (run-tests "test"))
  ([param] (reload/reload)                            ; clj-reload, then eftest
           (eftest/run-tests (eftest/find-tests param) ...)))
```

## 2. Run tests (the important part)

The shape is: reload, run, then read the summary.

Prefer the project's own runner when it has one — it handles discovery and
reloading the way the project expects:

```bash
# whole namespace
nreplctl eval "(user/run-tests 'myapp.game.commands-test)"
# a single test var
nreplctl eval "(user/run-tests 'myapp.game.commands-test/handles-double)"
```

Otherwise drive `clojure.test` directly (reload so it sees your latest edits):

```bash
nreplctl eval "(require 'myapp.game.commands-test :reload) (clojure.test/run-tests 'myapp.game.commands-test)"
# one var: (clojure.test/test-vars [#'myapp.game.commands-test/handles-double])
```

**Read the result — the exit code does not tell you whether tests passed.** A
failing assertion prints a `FAIL`/`ERROR` report but is *not* an exception, so
**nreplctl exits `0` even when tests fail**. Both `clojure.test` and eftest
return a summary map; judge success from `:fail` and `:error`:

```
{:test 4, :pass 19, :fail 0, :error 0, :type :summary}   ; green
{:test 1, :pass 0,  :fail 1, :error 0, :type :summary}   ; RED — one failure, yet exit 0
```

So green = `:fail 0` and `:error 0`. Exit `1` means the eval itself blew up —
usually the test namespace failed to compile, which is a real bug to fix, not a
test failure. Exit `2` means you never reached the server (see step 1).
`test-vars` prints failures but returns `nil`, so use `run-tests` when you want a
summary to parse.

## 3. Reload changed code before re-testing

`require` caches: a second `require` without `:reload` is a no-op, so tests would
run against the **old** code and give a misleading pass or fail. After editing a
file, reload before testing:

- Project runners like the one above already reload for you — one more reason to
  prefer them.
- Otherwise: `:reload` (that namespace) or `:reload-all` (it and its deps) on the
  `require`, or the project's `(user/reset)` / `(reload/reload)` /
  tools.namespace `refresh`. For integrant projects, `reset` reloads *and*
  restarts the system so live state matches the new code.

## 4. Eval arbitrary code

Quoting is the main friction: Clojure uses both `"` (strings) and `'` (the quote
reader macro, e.g. `'my.ns`), and so does the shell. Three reliable options:

- **Single-quote the shell arg, write Clojure quotes as `(quote sym)`** (no `'`
  inside): `nreplctl eval '(require (quote clojure.string)) (clojure.string/blank? "")'`
- **Double-quote the shell arg, use Clojure `'` freely, escape inner `\"`**:
  `nreplctl eval "(require 'clojure.string) (clojure.string/blank? \"\")"`
- **For anything hairy, write it to a temp file** and pass its contents, which
  sidesteps quoting entirely:
  `nreplctl eval "$(cat /tmp/snippet.clj)"`

Evaluation runs in `user`. To work inside a namespace, fully-qualify symbols, or
put `(in-ns 'my.ns)` as the first form in the *same* eval string. Because the
runtime is shared, you can inspect the live system across calls, e.g.
`(keys integrant.repl.state/system)` to see the running components, or call a
project fn against the warm DB pool.

## 5. Long or hanging evals: `--timeout`

`-t/--timeout SECONDS` bounds each read; on expiry nreplctl prints `Eval timed
out after Ns` and exits `1`. Use it for a suite that might hang. The eval keeps
running server-side after a timeout, so the result is lost — re-run narrower if
you need it. Leave it off (the default) for normal runs.

## Gotchas

- **Stale `.nrepl-port`** — exit `2` usually means the file points at a dead
  port, not that the REPL is down. Find the live port and use `--port`.
- **Exit `0` ≠ tests passed** — always read `:fail`/`:error` from the summary.
  Only a thrown exception (a broken test ns) makes the exit non-zero.
- **Stale code** — reload before re-testing, or the pass/fail describes old code.
- **Current namespace resets** — every call starts in `user`; `def`s and
  `require`s persist, but `(in-ns ...)` does not carry across calls.
- **Quoting** — single-quote the arg with `(quote sym)`, or double-quote and
  escape `\"`; for big snippets use a temp file and `"$(cat …)"`.
- **Non-JVM nREPLs** — nreplctl also speaks to babashka and let-go servers, which
  report one value per form (a leading `(require …)` prints `nil` first) rather
  than only the last. Read the whole output, not just the last line.
