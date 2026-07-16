# nrepl-client Library Extraction Implementation Plan

> **For agentic workers:** Use executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Move the nREPL client out of nreplctl into the standalone `nrepl-client` library (namespace `nrepl-client.core`), and repoint nreplctl at it as its first consumer via a `:local/root` dependency.

**Tech Stack:** let-go (`.lg`) + lgx across two repos — `nrepl-client` (the new library) and `nreplctl` (the CLI consumer). The client depends on let-go's `net`/`bencode` namespaces, which are not yet in a released `lg`, so both repos build and test against the patched lg via `LGX_LG=/Users/andrew/Projects/let-go/lg` until the upstream release.

---

## Design

**Why.** The nREPL client already lives in one namespace (`nreplctl.nrepl`) and is session-oriented (clone once, eval many, close separately). Extracting it is a lift-and-rename, not a rewrite. It lets other let-go programs reuse the client (the maintainer wants it for legmacs) and leaves nreplctl a thin CLI consumer.

**What moves.** `nreplctl.nrepl` becomes `nrepl-client.core`, verbatim except the `ns` form, with its tests. `resolve-port`, `msg-effects`, `connect!`, `clone!`, `eval!`, `close!` keep their names and signatures. `resolve-port` stays in the library, since discovering the port from `.nrepl-port` is useful to any client.

**Dependency wiring.** nreplctl references the library with `:deps {nrepl-client {:local/root "../nrepl-client"}}`. The two repos work together locally (via `LGX_LG`) with no push, and it is a one-line swap to `:git/tag "v0.1.0"` when the library is published at the net/bencode release (lgx forbids mixing `:local/root` with `:git/*`, so it is one or the other). A local-path dep does not resolve in nreplctl's own CI; that is acceptable while everything is already pending the net/bencode release.

**Test split.** The library owns the client tests: `core_test.lg` (the `resolve-port` and `msg-effects` units) and `integration_test.lg` (end-to-end clone/eval/close against a spawned `lg -n`). nreplctl keeps a lean `integration_test.lg` that asserts `eval-cmd`'s exit codes (0/1/2) against a live server — its own orchestration logic, which the library cannot cover. `eval-cmd` returns the code as an int, so this needs no stdout capture.

**Library scaffolding.** The `nrepl-client` repo is a fresh lgx library template (no binary target). It needs a `.clj-kondo/config.edn` (the same `catch` plus `os`/`net`/`bencode` handling nreplctl uses), a `checks.yml` with the temporary patched-lg build and `LGX_LG` (net/bencode is not in a released lg), and a README for the client API. Its `release.yml` (tag → checks → GitHub Release) works as-is for a pure-`.lg` library.

**Repos and branches.** nreplctl is on `master` (clean, the full implementation is committed); do the nreplctl changes on a new branch, `feat/extract-nrepl-client`. The `nrepl-client` repo is greenfield — commit its initial library content directly to `master` (do not branch).

## File Structure

**nrepl-client** (`/Users/andrew/Projects/nrepl-client`):
- Create `src/nrepl_client/core.lg` — the client, from `nreplctl.nrepl` (ns `nrepl-client.core`). Replaces the placeholder `greet`.
- Create `test/nrepl_client/core_test.lg` — units, from `nreplctl.nrepl-test`. Replaces the placeholder greet test.
- Create `test/nrepl_client/integration_test.lg` — end-to-end, from `nreplctl.integration-test`.
- Create `.clj-kondo/config.edn` — `catch` + `os`/`net`/`bencode` handling.
- Modify `.github/workflows/checks.yml` — temporary patched-lg build + `LGX_LG`.
- Modify `README.md` — library description, API, usage, interim note.

**nreplctl** (`/Users/andrew/Projects/nreplctl`):
- Modify `lgx.edn` — add `:deps {nrepl-client {:local/root "../nrepl-client"}}`.
- Modify `src/nreplctl/core.lg` — require `nrepl-client.core` instead of `nreplctl.nrepl`.
- Modify `.clj-kondo/config.edn` — drop the moved namespaces; drop `net`/`bencode` from the exclude (keep `os`).
- Delete `src/nreplctl/nrepl.lg`, `test/nreplctl/nrepl_test.lg`, `test/nreplctl/integration_test.lg`.
- Create `test/nreplctl/integration_test.lg` — new CLI test asserting `eval-cmd` exit codes.

---

### Task 1: Populate the nrepl-client library

Repo: `/Users/andrew/Projects/nrepl-client` (commit to `master`).

**Files:**
- Create: `src/nrepl_client/core.lg`
- Create: `test/nrepl_client/core_test.lg`
- Create: `test/nrepl_client/integration_test.lg`
- Create: `.clj-kondo/config.edn`

- [ ] **Step 1: Move the client namespace**
  Copy `nreplctl/src/nreplctl/nrepl.lg` to `src/nrepl_client/core.lg`, changing only the `ns` form to `(ns nrepl-client.core "<keep the docstring>" (:require [clojure.string :as str]))`. Everything below the `ns` form is verbatim (`resolve-port`, `msg-effects`, `clone-timeout-ms`, `close-timeout-ms`, `connect!`, `no-nrepl!`, `clone!`, `eval!`, `close!`).

- [ ] **Step 2: Move the unit tests**
  Copy `nreplctl/test/nreplctl/nrepl_test.lg` to `test/nrepl_client/core_test.lg`; change the ns to `nrepl-client.core-test` and the require `[nreplctl.nrepl :as nrepl]` to `[nrepl-client.core :as nrepl]`. Keep all 14 deftests. Overwrite the placeholder greet test that shipped with the template.

- [ ] **Step 3: Move the integration test**
  Copy `nreplctl/test/nreplctl/integration_test.lg` to `test/nrepl_client/integration_test.lg`; change the ns to `nrepl-client.integration-test` and the require to `[nrepl-client.core :as nrepl]`. Verbatim otherwise (helpers `lg-binary`, `sh-single-quote`, `start-server!`, `stop-server!`, `wait-connect!`, `collect-eval`, and the `end-to-end-eval` deftest).

- [ ] **Step 4: clj-kondo config**
  Create `.clj-kondo/config.edn` mirroring nreplctl's: `:linters {:unresolved-namespace {:exclude [os net bencode]}}` plus `:config-in-ns` disabling `:unresolved-symbol`, `:unused-binding`, `:syntax` for `nrepl-client.core`, `nrepl-client.core-test`, `nrepl-client.integration-test`.

- [ ] **Step 5: Verify**
  Run `LGX_LG=/Users/andrew/Projects/let-go/lg lgx check`.
  Expected: fmt + lint clean; 15 tests (14 unit + 1 integration), 0 failures; no orphan `lg`/`tail` processes after.

- [ ] **Step 6: Commit** (in nrepl-client)
  `git commit -m "feat: nREPL client library extracted from nreplctl"`

### Task 2: nrepl-client CI and README

Repo: `/Users/andrew/Projects/nrepl-client`.

**Files:**
- Modify: `.github/workflows/checks.yml`
- Modify: `README.md`

- [ ] **Step 1: CI patched-lg step**
  Update `checks.yml` to mirror nreplctl's: after `jdx/mise-action@v3`, add `actions/setup-go@v5` (`go-version: '1.26'`) and a `Build patched lg` step that clones `--depth 1 -b tcp-client https://github.com/abogoyavlensky/let-go /tmp/let-go` and runs `go build -o /tmp/let-go/lg .`, then run the checks step as `lgx check` with `env: LGX_LG: /tmp/let-go/lg`. (This replaces the template's separate fmt/tests steps so lint runs too.) Keep the `TEMPORARY until a let-go release ships net/bencode` comment.

- [ ] **Step 2: README**
  Rewrite `README.md`: what the library is (a small nREPL client for let-go, bencode over TCP), the API (`connect!`, `clone!`, `eval!`, `close!`, `msg-effects`, `resolve-port` — one line each), a short usage example (`(require '[nrepl-client.core :as nrepl])` then dial/clone/eval/close), how to depend on it in `lgx.edn` (`:git/url` + `:git/tag`, and `:local/root` for local dev), and the temporary "build patched lg / `LGX_LG`" note until net/bencode ships in a release.

- [ ] **Step 3: Verify**
  `LGX_LG=/Users/andrew/Projects/let-go/lg lgx check` still green; `python3 -c "import yaml; yaml.safe_load(open('.github/workflows/checks.yml')); print('yaml ok')"`.

- [ ] **Step 4: Commit** (in nrepl-client)
  `git commit -m "ci: build patched lg until let-go release; document the client API"`

### Task 3: Repoint nreplctl at the library

Repo: `/Users/andrew/Projects/nreplctl`.

**Files:**
- Modify: `lgx.edn`, `src/nreplctl/core.lg`, `.clj-kondo/config.edn`
- Delete: `src/nreplctl/nrepl.lg`, `test/nreplctl/nrepl_test.lg`, `test/nreplctl/integration_test.lg`

- [ ] **Step 1: Branch**
  `git checkout -b feat/extract-nrepl-client` (master is the default branch).

- [ ] **Step 2: Add the dependency**
  In `lgx.edn`, add to `:deps` alongside `tiny-cli`: `nrepl-client {:local/root "../nrepl-client"}`.

- [ ] **Step 3: Swap the require**
  In `src/nreplctl/core.lg`, change `(:require [nreplctl.nrepl :as nrepl])` to `(:require [nrepl-client.core :as nrepl])`. No other edits — every `nrepl/…` call site is unchanged.

- [ ] **Step 4: Delete the in-tree client and its tests**
  `git rm src/nreplctl/nrepl.lg test/nreplctl/nrepl_test.lg test/nreplctl/integration_test.lg`.

- [ ] **Step 5: Prune clj-kondo**
  In `.clj-kondo/config.edn`, set `:unresolved-namespace {:exclude [os]}` (nreplctl's own source no longer references `net`/`bencode`). In `:config-in-ns`, keep `nreplctl.core`; remove `nreplctl.nrepl` and `nreplctl.nrepl-test`. (The `nreplctl.integration-test` entry returns in Task 4.)

- [ ] **Step 6: Verify the CLI still works via the library**
  Start a server in the background: `tail -f /dev/null | /Users/andrew/Projects/let-go/lg -n -p 17600`. Then `LGX_LG=/Users/andrew/Projects/let-go/lg lgx run -- eval --port 17600 '(+ 1 1)'` prints `2`, exit 0 (proves the `:local/root` dep resolves and `nrepl-client.core` loads). Kill the server. Also run `LGX_LG=/Users/andrew/Projects/let-go/lg lgx check` — fmt + lint clean, tests run (nreplctl has none until Task 4).

- [ ] **Step 7: Commit**
  `git commit -m "refactor: consume nrepl-client library; drop the in-tree client"`

### Task 4: nreplctl CLI integration test

Repo: `/Users/andrew/Projects/nreplctl` (same branch).

**Files:**
- Create: `test/nreplctl/integration_test.lg`
- Modify: `.clj-kondo/config.edn`

- [ ] **Step 1: Write the CLI exit-code test**
  Create `test/nreplctl/integration_test.lg`: ns `nreplctl.integration-test`, require `[clojure.test :refer [deftest is]]`, `[clojure.string :as str]`, `[nrepl-client.core :as nrepl]`, `[nreplctl.core :as core]`. (The `[nrepl-client.core :as nrepl]` require is needed because the copied `wait-connect!` helper calls `nrepl/connect!`; it resolves through nreplctl's `:local/root` dep.) Copy the generic server-spawn helpers (`lg-binary`, `sh-single-quote`, `start-server!`, `stop-server!`, `wait-connect!`) from the library's integration test. One `deftest` that spawns a server, `wait-connect!`s (then closes that probe connection with `(nrepl/close! conn nil)` so the test leaves no open socket), then asserts on `eval-cmd`'s int return (it returns the exit code without exiting the process):
  - `(core/eval-cmd {:args {:code "(+ 1 1)"} :opts {:port (str port) :port-file ".nrepl-port" :host "127.0.0.1"}})` returns `0`.
  - the same with `:code "(undefined-fn-xyz)"` returns `1`.
  - `{:args {:code "(+ 1 1)"} :opts {:port nil :port-file "<a guaranteed-absent path under (os/temp-dir)>" :host "127.0.0.1"}}` returns `2` (no port resolvable).
  Kill the server in a `finally`. Note the ctx shape `eval-cmd` expects: `{:args {:code <str>} :opts {:port <str-or-nil> :port-file <str> :host <str> :timeout <str-or-nil>}}`.

- [ ] **Step 2: clj-kondo**
  Re-add `nreplctl.integration-test` to `:config-in-ns` in `.clj-kondo/config.edn` (it uses `catch` and `os`).

- [ ] **Step 3: Verify**
  Run `LGX_LG=/Users/andrew/Projects/let-go/lg lgx check`.
  Expected: fmt + lint clean; the CLI integration test passes (eval-cmd returns 0/1/2); 0 failures; no orphan processes.

- [ ] **Step 4: Commit**
  `git commit -m "test: CLI integration test asserting eval-cmd exit codes"`

## Final verification

- [ ] `nrepl-client`: `LGX_LG=/Users/andrew/Projects/let-go/lg lgx check` green (15 tests).
- [ ] `nreplctl`: `LGX_LG=/Users/andrew/Projects/let-go/lg lgx check` green (CLI integration test).
- [ ] End-to-end once more: with a live server, `LGX_LG=… lgx run -- eval --port <p> '(+ 1 1)'` prints `2`.

## Release follow-up (at the net/bencode let-go release — outside this plan)

- `nrepl-client`: bump `.mise.toml` lg to the release; drop the temporary "Build patched lg" step from `checks.yml`; `lgx release 0.1.0` (tag + push `v0.1.0`).
- `nreplctl`: switch the dep from `:local/root "../nrepl-client"` to `:git/url "https://github.com/abogoyavlensky/nrepl-client"` + `:git/tag "v0.1.0"`; bump `.mise.toml` lg; drop its temporary patched-lg CI step.
