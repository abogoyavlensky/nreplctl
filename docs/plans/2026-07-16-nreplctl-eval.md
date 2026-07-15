# nreplctl eval Implementation Plan

> **For agentic workers:** Use executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** A let-go CLI tool that sends Clojure code to a running nREPL server and prints the results, with friendly failures — plus the minimal TCP/bencode primitives it needs added to let-go upstream.

**Tech Stack:** let-go (.lg) + tiny-cli for the tool; Go (`pkg/rt`) for the new let-go namespaces; `zeebo/bencode` (already vendored in let-go).

---

## Design

### Why two repos

let-go 1.11.1 has no TCP client in its lg-facing stdlib — only AF_UNIX sockets (`unix/connect`), HTTP, and pods over stdin/stdout. nREPL is bencode-over-TCP, so the tool cannot exist in pure `.lg` today. The work therefore splits:

- **Phase 1 — let-go** (`/Users/andrew/Projects/let-go`, branch `tcp-client`, already created): add two small generic namespaces in Go — `net` (TCP client primitives) and `bencode` (message framing over a conn, reusing the already-vendored `zeebo/bencode`, the same library lg's own nREPL server uses).
- **Phase 2 — nreplctl** (this repo): all nREPL semantics — port discovery, session clone, the eval loop, output routing, exit codes — in pure `.lg`.

**Phase 2 does NOT wait for a new let-go release.** Develop and test against the locally built binary at `/Users/andrew/Projects/let-go/lg` by setting `LGX_LG=/Users/andrew/Projects/let-go/lg` (lgx honors this env var when invoking lg). `.mise.toml` keeps lg 1.11.1 for now; bump it once a release ships with `net`/`bencode`, then drop the temporary CI build step (Task 8).

### New let-go namespaces (the contract both repos share)

`net` — portable (no build tag), pattern-matched to `pkg/rt/unix_linux.go` (Boxed handles, `ns.Def`, errors as `fmt.Errorf` with the fn name prefix):

```clojure
(net/dial host port)        ; → conn (Boxed). 3s connect timeout. Throws on failure.
(net/close! conn)           ; → nil
(net/write! conn str-or-bytes) ; → nil (raw, for generality)
(net/read! conn max-bytes)  ; → byte-array, nil on EOF (raw, for generality)
```

`bencode` — operates on a `net` conn; the conn's Boxed value holds `{net.Conn, *bencode.Decoder}` so buffered bytes survive between reads:

```clojure
(bencode/write! conn value)          ; → nil. Encodes one bencode value.
(bencode/read! conn)                 ; → decoded value, nil on clean EOF. Blocks.
(bencode/read! conn timeout-ms)      ; same, but errors with a message containing
                                     ; "timeout" if no message arrives in time
                                     ; (via SetReadDeadline; clear deadline after).
```

Value conversion, pinned both directions:

| lg → bencode | bencode → lg |
|---|---|
| String → byte string | byte string → String |
| keyword → its name (`:op` → `"op"`) | int → Int |
| Int → int | list → vector |
| vector/list → list | dict → map with String keys |
| map (String/keyword keys) → dict | |
| anything else (nil, bool, float) → error | |

### nreplctl CLI shape

```
nreplctl eval '<code>' [-p/--port N] [--port-file PATH] [--host HOST] [-t/--timeout SECONDS]
```

- `--port` (explicit, wins) → else read `--port-file` (default `.nrepl-port` in cwd), trim, `parse-long`.
- `--host` defaults to `127.0.0.1`.
- `--timeout` (seconds, off by default) bounds each read during eval; on expiry: `Eval timed out after Ns` to stderr, exit 1. The `clone` handshake always uses a fixed 5s read timeout so a non-nREPL listener can't hang the tool.
- Code is one required positional. Multiple forms in one string are sent as-is; the server evaluates them all.

### Eval flow (standard ops only — works against `lg -n`, JVM nREPL, babashka)

1. `clone` → `{"op" "clone" "id" "1"}`, read until `done`, keep `new-session`.
2. `eval` → `{"op" "eval" "code" code "session" s "id" "2"}`.
3. Loop reading messages until `status` contains `done`:
   - `"out"` → stdout as it arrives (no added newline — chunks carry their own)
   - `"err"` → stderr (`binding [*out* *err*]`)
   - `"value"` → stdout, each on its own line (every value, not just the last — REPL-faithful and better for agents)
   - error if `status` contains `"eval-error"` OR the message has `"ex"` (lg's server sends `ex` without the status)
4. After all evals: one `close` op for the session (reading its `done` response with a short timeout so no stray bytes are left), then `net/close!`. Session close lives *outside* `eval!` so one session can serve several evals without unread responses corrupting the stream.

Abnormal termination is a connection failure (exit 2), with a friendly message:
- `bencode/read!` returns nil (EOF) before `done` → `Connection closed by nREPL server before eval completed`.
- `clone` times out, errors, or returns no `new-session` → `No nREPL response from HOST:PORT — is that a real nREPL server?`

### Exit codes and friendly failures (agent-facing contract)

| Code | Meaning | Example stderr message |
|---|---|---|
| 0 | eval succeeded | |
| 1 | code threw, or eval timed out | server's `err` output / `Eval timed out after 30s` |
| 2 | can't resolve port, connect, or complete the protocol | `No .nrepl-port file found and no --port given — is an nREPL server running here?` / `Could not connect to nREPL at 127.0.0.1:2137 — is the server still running?` / the EOF and clone-failure messages above |

All nreplctl diagnostics go to stderr; stdout carries only evaluated output and values.

### Testing strategy

- **let-go:** Go tests in `pkg/rt/net_test.go` — dial/read/write against a `net.Listener` opened in the test; bencode round-trips through a real TCP pair; conversion edge cases; timeout behavior.
- **nreplctl unit (.lg):** `resolve-port` against temp files; `msg-effects` (pure message → actions fn) with no sockets involved.
- **nreplctl integration (.lg):** spawn the same lg under test (`LGX_LG` or `lg`) as `lg -n -p <free-port>` in the background, run a real clone/eval/close round-trip, kill it in a finally.
- **Manual cross-impl check:** babashka is installed via mise — verify against `bb nrepl-server`.

## File Structure

**let-go** (`/Users/andrew/Projects/let-go`, branch `tcp-client`):
- Create: `pkg/rt/net.go` — `net` + `bencode` namespaces, conn handle struct, value conversion helpers (~150–200 lines)
- Create: `pkg/rt/net_test.go` — Go tests for both namespaces

**nreplctl** (this repo):
- Create: `src/nreplctl/nrepl.lg` — `resolve-port`, `connect!`, `clone!`, `eval!`, `msg-effects` (pure)
- Modify: `src/nreplctl/core.lg` — replace `greet` with `eval-cmd` (orchestration, error → message → exit-code mapping)
- Modify: `main.lg` — tiny-cli spec for the `eval` command
- Create: `test/nreplctl/nrepl_test.lg` — unit tests
- Create: `test/nreplctl/integration_test.lg` — end-to-end test
- Delete: `test/nreplctl/core_test.lg` (greet test goes away with greet)
- Modify: `.github/workflows/checks.yml` — temporary step building lg from the fork branch
- Modify: `README.md` — usage

---

### Task 1: `net` namespace in let-go

Work in `/Users/andrew/Projects/let-go` on branch `tcp-client`.

**Files:**
- Create: `pkg/rt/net.go`
- Test: `pkg/rt/net_test.go`

- [ ] **Step 1: Write the failing test**
  In `net_test.go`: start a `net.Listener` on `127.0.0.1:0`; from lg code (follow the eval-helper conventions used by existing `pkg/rt` tests, e.g. `http_test.go`) call `net/dial` with the listener's host/port, `net/write!` a string and a byte-array (with a >127 byte, verbatim on the wire), assert the server side received both; have the server write bytes back, assert `net/read!` returns them as a byte-array and returns nil after the server closes; `net/close!` succeeds. Also: `net/dial` to a closed port returns an error mentioning `net/dial`.

- [ ] **Step 2: Run test to verify it fails**
  Run: `go test ./pkg/rt -run TestNet -count=1`
  Expected: FAIL (unresolved `net/dial`)

- [ ] **Step 3: Implement `pkg/rt/net.go`**
  Follow `unix_linux.go` structurally: `RegisterInstaller` in `init`, `vm.NativeFnType.Wrap` fns, Boxed conn. No build tag — plain TCP is portable. The Boxed value is a small struct holding the `net.Conn` plus a lazily-created `*bencode.Decoder` (used in Task 2). `dial` uses `net.DialTimeout("tcp", addr, 3*time.Second)`; accept port as Int, host as String.

- [ ] **Step 4: Run test to verify it passes**
  Run: `go test ./pkg/rt -run TestNet -count=1`
  Expected: PASS

- [ ] **Step 5: Commit** (in let-go)
  `git commit -m "rt: add net namespace - minimal TCP client primitives"`

### Task 2: `bencode` namespace in let-go

**Files:**
- Modify: `pkg/rt/net.go` (or a sibling `bencode.go` if it reads better)
- Test: `pkg/rt/net_test.go`

- [ ] **Step 1: Write the failing test**
  Tests over a real TCP pair: `bencode/write!` an nREPL-shaped map `{"op" "eval" "code" "(+ 1 1)"}` from lg, decode on the Go side and assert the dict; encode a response dict with nested list (`"status" ["done"]`) on the Go side, assert `bencode/read!` yields `{"status" ["done"] ...}` as an lg map with String keys and a vector value. Conversion cases: keyword keys become name strings, Int round-trips, nil/bool/float values error on write. Timeout case: `(bencode/read! conn 100)` against a silent server errors with a message containing `timeout`, and the conn still works for a subsequent read (deadline cleared).

- [ ] **Step 2: Run test to verify it fails**
  Run: `go test ./pkg/rt -run TestBencode -count=1`
  Expected: FAIL

- [ ] **Step 3: Implement**
  `bencode/write!`: convert lg value → Go (`map[string]any`/`[]any`/`string`/`int64`) per the conversion table, `bencode.EncodeBytes`, write to conn. `bencode/read!`: reuse the conn's persistent `*bencode.Decoder` (create on first use), decode into `any`, convert → lg values per the table. Optional second arg (Int, ms): `SetReadDeadline(now+ms)` before the decode, reset to zero after; map `net.Error.Timeout()` to an error whose message contains `timeout`. Clean EOF → nil.

- [ ] **Step 4: Run test to verify it passes**
  Run: `go test ./pkg/rt -run 'TestNet|TestBencode' -count=1`
  Expected: PASS

- [ ] **Step 5: Full rt test suite**
  Run: `go test ./pkg/rt -count=1`
  Expected: PASS (no regressions)

- [ ] **Step 6: Commit** (in let-go)
  `git commit -m "rt: add bencode namespace - message framing over net conns"`

### Task 3: Build local lg and smoke-test end to end

**Files:** none (build artifact only)

- [ ] **Step 1: Build**
  Run in `/Users/andrew/Projects/let-go`: `go build -o lg .`
  Expected: `/Users/andrew/Projects/let-go/lg` exists; `./lg -v` prints a version.

- [ ] **Step 2: Smoke-test against lg's own nREPL server**
  Start `./lg -n -p 17423 -e '(sleep 60000)' &` (or similar keep-alive), then:
  `./lg -e '(let [c (net/dial "127.0.0.1" 17423)] (bencode/write! c {"op" "clone" "id" "1"}) (println (bencode/read! c 5000)) (net/close! c))'`
  Expected: a printed map containing `"new-session"`. Kill the server after.

- [ ] **Step 3: Push the branch** (needed by Task 8's CI step)
  `git push -u origin tcp-client`

### Task 4: `resolve-port` and `msg-effects` in nreplctl

All nreplctl commands from here on run with the local lg: prefix with `LGX_LG=/Users/andrew/Projects/let-go/lg`.

**Files:**
- Create: `src/nreplctl/nrepl.lg`
- Test: `test/nreplctl/nrepl_test.lg`

- [ ] **Step 1: Write the failing tests**
  `resolve-port`: takes `{:port <str-or-nil> :port-file <path>}` → int, or throws `ex-info` whose message is the friendly text from the design (distinguish: no file+no port / unreadable port value / bad `--port` value). Cases: explicit `:port` wins over an existing file; port file with `"2137\n"` (write via `spit` to a file under `os/temp-dir`) parses; missing file + no port throws the "No .nrepl-port file found" message.
  `msg-effects`: pure fn, nREPL message map → `{:events [[:out s] | [:err s] | [:value s] ...] :done? bool :error? bool}`. Cases: out-only msg; err msg; value msg; `{"status" ["done"]}` → done; `{"status" ["eval-error"] ...}` → error; msg with `"ex"` → error; combined value+done msg.

- [ ] **Step 2: Run tests to verify they fail**
  Run: `LGX_LG=/Users/andrew/Projects/let-go/lg lgx test`
  Expected: FAIL (namespace `nreplctl.nrepl` missing)

- [ ] **Step 3: Implement both fns in `nrepl.lg`**
  Pure logic only in `msg-effects`; `resolve-port` does `slurp` + `trim` + `parse-long`. Keep the friendly messages as the ex-info messages so `core.lg` just prints them.

- [ ] **Step 4: Run tests to verify they pass**
  Run: `LGX_LG=/Users/andrew/Projects/let-go/lg lgx test`
  Expected: PASS

- [ ] **Step 5: Commit**
  `git commit -m "feat: add port resolution and nREPL message handling"`

### Task 5: nREPL client fns

**Files:**
- Modify: `src/nreplctl/nrepl.lg`

- [ ] **Step 1: Implement `connect!`, `clone!`, `eval!`, `close!`**
  Thin IO over the Task 1/2 primitives (integration-tested in Task 7, no unit tests):
  - `(connect! host port)` → conn; let `net/dial` errors propagate (core maps them).
  - `(clone! conn)` → session id string; reads with the fixed 5000ms timeout until `done`. Timeout, nil (EOF), or a response without `new-session` → throw `ex-info` with the "No nREPL response from HOST:PORT…" message.
  - `(eval! conn session code {:timeout-ms n-or-nil :emit f})` → `{:error? bool :timeout? bool}`. Sends the eval op, loops `bencode/read!` (passing `:timeout-ms` when set), feeds each message through `msg-effects`, calls `emit` with each event as it arrives (streaming — don't batch), stops on done. A read error containing `timeout` → `{:timeout? true}`. A nil read (EOF) before done → throw `ex-info` with the "Connection closed by nREPL server…" message. Does NOT send `close` — the same session must support several evals without leaving unread responses on the stream.
  - `(close! conn session)` → nil. Sends the `close` op, best-effort reads its response with a short (1000ms) timeout swallowing errors, then `net/close!`. Called once by `eval-cmd` in its finally.
  - Message ids: fixed `"1"`/`"2"`/`"3"` strings are fine for a one-shot client.

- [ ] **Step 2: Verify it loads**
  Run: `LGX_LG=/Users/andrew/Projects/let-go/lg lgx test`
  Expected: PASS (existing tests still green, namespace compiles)

- [ ] **Step 3: Commit**
  `git commit -m "feat: add nREPL client - connect, clone, eval loop"`

### Task 6: CLI wiring — `eval` command

**Files:**
- Modify: `src/nreplctl/core.lg`, `main.lg`
- Delete: `test/nreplctl/core_test.lg`

- [ ] **Step 1: Implement `eval-cmd` in `core.lg`**
  Replace `greet`. `(eval-cmd ctx)` → exit code int (testable); a thin `(eval-cmd! ctx)` wrapper calls `(os/exit ...)` with it. Flow: resolve port (ex-info → print message to stderr, 2) → connect (net/dial error → the "Could not connect…" message with host:port, 2) → clone (ex-info → its message to stderr, 2) → eval with the default emit (`[:out s]` → `(print s)`, `[:err s]` → print to stderr via `binding [*out* *err*]`, `[:value s]` → `(println s)`); an EOF ex-info from `eval!` → its message to stderr, 2 → `close!` in a finally → 1 if `:error?` or `:timeout?` (print the timeout message with the configured seconds), else 0.

- [ ] **Step 2: Wire the tiny-cli spec in `main.lg`**
  Command `eval`, doc string, one required positional `:code`. Command opts (`:value? true` each except none are flags): `:port` (`-p`/`--port`), `:port-file` (`--port-file`, `:default ".nrepl-port"`), `:host` (`--host`, `:default "127.0.0.1"`), `:timeout` (`-t`/`--timeout`, seconds). Validate `:port` and `:timeout` as positive integers via `:validate`. App `:doc`: one line about evaluating code in a running nREPL server. Remove the `greet` command and its test file.

- [ ] **Step 3: Manual verification against the local lg server**
  Start: `/Users/andrew/Projects/let-go/lg -n -p 17423 -e '(sleep 60000)' &`
  - `LGX_LG=/Users/andrew/Projects/let-go/lg lgx run -- eval --port 17423 '(+ 1 1)'` → prints `2`, exit 0
  - `... eval --port 17423 '(require (quote [string :as s]))(s/join ", " ["a" "b" "c"])'` → prints `nil` then `"a, b, c"`
  - `... eval --port 17423 '(println "hi")'` → prints `hi` then `nil`
  - `... eval --port 17423 '(undefined-fn)'` → error text on stderr, exit 1 (`echo $?`)
  - `... eval --port 1 '(+ 1 1)'` → "Could not connect…" on stderr, exit 2
  - no server file case: run in a dir without `.nrepl-port`, no `--port` → friendly message, exit 2
  - `echo 17423 > .nrepl-port` then `... eval '(+ 1 1)'` → `2` (file discovery works)
  Kill the server after.

- [ ] **Step 4: Commit**
  `git commit -m "feat: wire eval command with port discovery and exit codes"`

### Task 7: Integration test

**Files:**
- Create: `test/nreplctl/integration_test.lg`

- [ ] **Step 1: Write the test**
  Helpers: lg binary = `(os/getenv "LGX_LG")` or `"lg"`; port = `(os/free-port)`; spawn via `(os/sh "sh" "-c" "<lg> -n -p <port> -e '(sleep 60000)' >/dev/null 2>&1 & echo $!")` capturing the pid; retry `connect!` in a loop (`(sleep 100)`, ~50 attempts) until the server answers. In the test body: clone once, then over the same session — `eval!` of `(+ 1 1)` with a collecting `:emit` (atom conj), assert events contain `[:value "2"]` and `:error?` false; eval of `(undefined-fn-xyz)` → `:error?` true; a third eval of `(+ 2 3)` → `[:value "5"]` (proves session/stream reuse stays clean); eval with `:timeout-ms 100` of `(sleep 5000)` → `:timeout? true` (run this last — the late reply may still arrive on the wire). Finish with `close!`. Kill the server pid in a `finally` (`(os/sh "kill" pid)`).

- [ ] **Step 2: Run the full suite**
  Run: `LGX_LG=/Users/andrew/Projects/let-go/lg lgx test`
  Expected: PASS

- [ ] **Step 3: Manual cross-impl check (babashka)**
  `bb nrepl-server 17424 &`, then `LGX_LG=... lgx run -- eval --port 17424 '(require (quote [clojure.string :as str]))(str/join ", " ["a" "b" "c"])'` → `nil` and `"a, b, c"`. Kill it after. If bb behaves differently (e.g. per-form value messages), confirm output still reads correctly — that's the point of the check.

- [ ] **Step 4: Commit**
  `git commit -m "test: add end-to-end integration test against live nREPL server"`

### Task 8: CI, lint, docs

**Files:**
- Modify: `.github/workflows/checks.yml`, `README.md`

- [ ] **Step 1: Temporary CI step for the patched lg**
  mise installs lg 1.11.1 (no `net`/`bencode`), so CI must build lg from the fork branch until a release ships. In `checks.yml`, before "Run all checks":

  ```yaml
  - uses: actions/setup-go@v5
    with:
      go-version: '1.26'
  # TEMPORARY until a let-go release ships net/bencode; then bump lg in
  # .mise.toml and delete this step + the LGX_LG env below.
  - name: Build patched lg
    run: |
      git clone --depth 1 -b tcp-client https://github.com/abogoyavlensky/let-go /tmp/let-go
      cd /tmp/let-go && go build -o /tmp/let-go/lg .
  ```
  and set `LGX_LG: /tmp/let-go/lg` as `env` on the "Run all checks" step.

- [ ] **Step 2: Full local check**
  Run: `LGX_LG=/Users/andrew/Projects/let-go/lg lgx check`
  Expected: fmt, lint, tests all PASS. If clj-kondo flags the `net`/`bencode` namespaces as unresolved, add them to a `.clj-kondo/config.edn` exclude rather than sprinkling ignores.

- [ ] **Step 3: Update README**
  Replace the template text: what nreplctl is, `eval` usage with the flag table, port discovery order, exit codes (the agent-facing table from the design), the require example, and a development note about `LGX_LG` while the let-go release is pending.

- [ ] **Step 4: Commit**
  `git commit -m "ci: build patched lg until let-go release; document usage"`

### Follow-up (outside this plan)

- PR the `tcp-client` branch to `nooga/let-go`; once a release ships, bump `lg` in `.mise.toml`, drop the CI build step and `LGX_LG` notes.
