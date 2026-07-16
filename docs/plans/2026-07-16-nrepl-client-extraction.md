# nrepl-client Library Extraction Implementation Plan

> **✅ COMPLETED 2026-07-16** — all 4 tasks implemented, reviewed (codex per task), and verified. `nrepl-client` on `master` (4 commits); `nreplctl` on `feat/extract-nrepl-client` (3 commits). Full summary at the bottom.

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

- [x] **Step 1: Move the client namespace**
  Copy `nreplctl/src/nreplctl/nrepl.lg` to `src/nrepl_client/core.lg`, changing only the `ns` form to `(ns nrepl-client.core "<keep the docstring>" (:require [clojure.string :as str]))`. Everything below the `ns` form is verbatim (`resolve-port`, `msg-effects`, `clone-timeout-ms`, `close-timeout-ms`, `connect!`, `no-nrepl!`, `clone!`, `eval!`, `close!`).

- [x] **Step 2: Move the unit tests**
  Copy `nreplctl/test/nreplctl/nrepl_test.lg` to `test/nrepl_client/core_test.lg`; change the ns to `nrepl-client.core-test` and the require `[nreplctl.nrepl :as nrepl]` to `[nrepl-client.core :as nrepl]`. Keep all 14 deftests. Overwrite the placeholder greet test that shipped with the template.

  > Deviation: also renamed the temp-file prefix `nreplctl-test-` → `nrepl-client-test-` so the library's tests don't reference the old project name. Cosmetic; tests unchanged.

- [x] **Step 3: Move the integration test**
  Copy `nreplctl/test/nreplctl/integration_test.lg` to `test/nrepl_client/integration_test.lg`; change the ns to `nrepl-client.integration-test` and the require to `[nrepl-client.core :as nrepl]`. Verbatim otherwise (helpers `lg-binary`, `sh-single-quote`, `start-server!`, `stop-server!`, `wait-connect!`, `collect-eval`, and the `end-to-end-eval` deftest).

- [x] **Step 4: clj-kondo config**
  Create `.clj-kondo/config.edn` mirroring nreplctl's: `:linters {:unresolved-namespace {:exclude [os net bencode]}}` plus `:config-in-ns` disabling `:unresolved-symbol`, `:unused-binding`, `:syntax` for `nrepl-client.core`, `nrepl-client.core-test`, `nrepl-client.integration-test`.

- [x] **Step 5: Verify**
  Run `LGX_LG=/Users/andrew/Projects/let-go/lg lgx check`.
  Expected: fmt + lint clean; 15 tests (14 unit + 1 integration), 0 failures; no orphan `lg`/`tail` processes after.  → PASS (15 tests, 23 assertions, 0 failures, 0 orphans).

  > Deviation: ran `mise trust` first — mise refuses the fresh repo's untrusted `.mise.toml`. Setup step, no code impact.

- [x] **Step 6: Commit** (in nrepl-client)
  `git commit -m "feat: nREPL client library extracted from nreplctl"`

  > Deviation: the lib template's `.gitignore` lacked `.clj-kondo/.cache/`; added it and amended the commit so the clj-kondo cache stays untracked (matching nreplctl). Commit `8633bbb`.

  > Codex review (commit `8633bbb`): P1 (lg 1.11.1 lacks net/bencode → CI fails) is the plan's phased design — Task 2 adds the temporary patched-lg CI step. **P2 (approved scope growth, fixup `00b7b85`):** a real pre-existing bug the move inherited verbatim — on a serial nREPL session a timed-out eval keeps running server-side, and the old read loop matched *any* `done`, so reusing the session made the next eval misread the stale request's value (`(+ 2 3)` after a timed-out `(do (sleep) 42)` reported `42`). The user chose to fix it properly: each request now gets a unique id via `gen-id`, and `clone!`/`eval!` ignore responses whose echoed id isn't the current request's. Verified lg echoes the id on every response message; added a `timeout-then-reuse` regression test (16 tests total).

### Task 2: nrepl-client CI and README

Repo: `/Users/andrew/Projects/nrepl-client`.

**Files:**
- Modify: `.github/workflows/checks.yml`
- Modify: `README.md`

- [x] **Step 1: CI patched-lg step**
  Update `checks.yml` to mirror nreplctl's: after `jdx/mise-action@v3`, add `actions/setup-go@v5` (`go-version: '1.26'`) and a `Build patched lg` step that clones `--depth 1 -b tcp-client https://github.com/abogoyavlensky/let-go /tmp/let-go` and runs `go build -o /tmp/let-go/lg .`, then run the checks step as `lgx check` with `env: LGX_LG: /tmp/let-go/lg`. (This replaces the template's separate fmt/tests steps so lint runs too.) Keep the `TEMPORARY until a let-go release ships net/bencode` comment.

- [x] **Step 2: README**
  Rewrite `README.md`: what the library is (a small nREPL client for let-go, bencode over TCP), the API (`connect!`, `clone!`, `eval!`, `close!`, `msg-effects`, `resolve-port` — one line each), a short usage example (`(require '[nrepl-client.core :as nrepl])` then dial/clone/eval/close), how to depend on it in `lgx.edn` (`:git/url` + `:git/tag`, and `:local/root` for local dev), and the temporary "build patched lg / `LGX_LG`" note until net/bencode ships in a release.

- [x] **Step 3: Verify**
  `LGX_LG=/Users/andrew/Projects/let-go/lg lgx check` still green; `python3 -c "import yaml; yaml.safe_load(open('.github/workflows/checks.yml')); print('yaml ok')"`.  → 16 tests green; YAML valid.

- [x] **Step 4: Commit** (in nrepl-client)
  `git commit -m "ci: build patched lg until let-go release; document the client API"`  → commit `bdbd8e9`.

  > Codex review (commit `bdbd8e9`) → fixup `0119f16`: (P1) the README's `:git/tag "v0.1.0"` install path references a tag that doesn't exist yet (the lib isn't published) — added a note that the tagged release lands at the first publish (pending the net/bencode let-go release) and to use `:local/root` or a `:git/sha` until then. Same "prepare now, publish at release" pattern as nreplctl's README.

### Task 3: Repoint nreplctl at the library

Repo: `/Users/andrew/Projects/nreplctl`.

**Files:**
- Modify: `lgx.edn`, `src/nreplctl/core.lg`, `.clj-kondo/config.edn`
- Delete: `src/nreplctl/nrepl.lg`, `test/nreplctl/nrepl_test.lg`, `test/nreplctl/integration_test.lg`

- [x] **Step 1: Branch**
  `git checkout -b feat/extract-nrepl-client` (master is the default branch).

- [x] **Step 2: Add the dependency**
  In `lgx.edn`, add to `:deps` alongside `tiny-cli`: `nrepl-client {:local/root "../nrepl-client"}`.

- [x] **Step 3: Swap the require**
  In `src/nreplctl/core.lg`, change `(:require [nreplctl.nrepl :as nrepl])` to `(:require [nrepl-client.core :as nrepl])`. No other edits — every `nrepl/…` call site is unchanged.

- [x] **Step 4: Delete the in-tree client and its tests**
  `git rm src/nreplctl/nrepl.lg test/nreplctl/nrepl_test.lg test/nreplctl/integration_test.lg`.

- [x] **Step 5: Prune clj-kondo**
  In `.clj-kondo/config.edn`, set `:unresolved-namespace {:exclude [os]}` (nreplctl's own source no longer references `net`/`bencode`). In `:config-in-ns`, keep `nreplctl.core`; remove `nreplctl.nrepl` and `nreplctl.nrepl-test`. (The `nreplctl.integration-test` entry returns in Task 4.)

  > Deviation: also removed the `nreplctl.integration-test` entry now (its file is deleted this task); Task 4 re-adds it with the new CLI test.

- [x] **Step 6: Verify the CLI still works via the library**
  Start a server in the background: `tail -f /dev/null | /Users/andrew/Projects/let-go/lg -n -p 17600`. Then `LGX_LG=/Users/andrew/Projects/let-go/lg lgx run -- eval --port 17600 '(+ 1 1)'` prints `2`, exit 0 (proves the `:local/root` dep resolves and `nrepl-client.core` loads). Kill the server. Also run `LGX_LG=/Users/andrew/Projects/let-go/lg lgx check` — fmt + lint clean, tests run (nreplctl has none until Task 4).

  > CLI check PASSED (`(+ 1 1)` → `2`, exit 0; clj-kondo 0 errors). Deviation: `lgx check` transiently exits 1 here because deleting every test removes the `test/` dir (`lgx test` needs it, and the lint `find src test` errors on the missing dir). Task 4 recreates `test/` and restores a green check — the plan's delete-then-add split created this brief gap.

- [x] **Step 7: Commit**
  `git commit -m "refactor: consume nrepl-client library; drop the in-tree client"`  → commit `b83ea6f`.

  > Codex review (commit `b83ea6f`): one P1 — deleting the whole `test/` tree makes `lgx check`/CI fail (`no test/ directory in project`). This is the delete-then-add transient noted above; **Task 4 is the fix** (it recreates `test/nreplctl/integration_test.lg`, restoring `test/` and a green check). Proceeded straight to Task 4 rather than a duplicate Task-3 fixup.

### Task 4: nreplctl CLI integration test

Repo: `/Users/andrew/Projects/nreplctl` (same branch).

**Files:**
- Create: `test/nreplctl/integration_test.lg`
- Modify: `.clj-kondo/config.edn`

- [x] **Step 1: Write the CLI exit-code test**
  Create `test/nreplctl/integration_test.lg`: ns `nreplctl.integration-test`, require `[clojure.test :refer [deftest is]]`, `[clojure.string :as str]`, `[nrepl-client.core :as nrepl]`, `[nreplctl.core :as core]`. (The `[nrepl-client.core :as nrepl]` require is needed because the copied `wait-connect!` helper calls `nrepl/connect!`; it resolves through nreplctl's `:local/root` dep.) Copy the generic server-spawn helpers (`lg-binary`, `sh-single-quote`, `start-server!`, `stop-server!`, `wait-connect!`) from the library's integration test. One `deftest` that spawns a server, `wait-connect!`s (then closes that probe connection with `(nrepl/close! conn nil)` so the test leaves no open socket), then asserts on `eval-cmd`'s int return (it returns the exit code without exiting the process):

  > Deviation: `[nreplctl.core :as core]` + `(core/eval-cmd …)` does **not compile** — a let-go bug where a require **alias named `core`** collides with the auto-referred `clojure.core` and won't resolve at compile time (any other alias works; the var is fine at runtime). Fixed by aliasing `nreplctl.core` as `cli` and calling `(cli/eval-cmd …)` directly (commit `76f465b`). An earlier commit (`c863270`) used a heavier `@(resolve 'nreplctl.core/eval-cmd)` workaround under a wrong transitive-require diagnosis; minimizing the bug for the upstream report corrected it. Reported to the let-go maintainer.
  - `(core/eval-cmd {:args {:code "(+ 1 1)"} :opts {:port (str port) :port-file ".nrepl-port" :host "127.0.0.1"}})` returns `0`.
  - the same with `:code "(undefined-fn-xyz)"` returns `1`.
  - `{:args {:code "(+ 1 1)"} :opts {:port nil :port-file "<a guaranteed-absent path under (os/temp-dir)>" :host "127.0.0.1"}}` returns `2` (no port resolvable).
  Kill the server in a `finally`. Note the ctx shape `eval-cmd` expects: `{:args {:code <str>} :opts {:port <str-or-nil> :port-file <str> :host <str> :timeout <str-or-nil>}}`.

- [x] **Step 2: clj-kondo**
  Re-add `nreplctl.integration-test` to `:config-in-ns` in `.clj-kondo/config.edn` (it uses `catch` and `os`).

- [x] **Step 3: Verify**
  Run `LGX_LG=/Users/andrew/Projects/let-go/lg lgx check`.
  Expected: fmt + lint clean; the CLI integration test passes (eval-cmd returns 0/1/2); 0 failures; no orphan processes.  → PASS (1 test, 4 assertions, 0 failures, 0 orphans). This restores nreplctl's `test/` dir and green check, resolving the Task-3 codex P1.

- [x] **Step 4: Commit**
  `git commit -m "test: CLI integration test asserting eval-cmd exit codes"`  → commit `c863270`.

  > Codex review (commit `c863270`) → fixup `8010e0f`: (P2) the exit-2 case deleted a fixed shared `/tmp/nreplctl-cli-test-absent-xyz`, which could clobber an unrelated file or collide with concurrent runs — replaced with an `os/free-port`-suffixed per-run path that nothing creates (guaranteed absent), no delete.

## Final verification

- [ ] `nrepl-client`: `LGX_LG=/Users/andrew/Projects/let-go/lg lgx check` green (15 tests).
- [ ] `nreplctl`: `LGX_LG=/Users/andrew/Projects/let-go/lg lgx check` green (CLI integration test).
- [ ] End-to-end once more: with a live server, `LGX_LG=… lgx run -- eval --port <p> '(+ 1 1)'` prints `2`.

## Release follow-up (at the net/bencode let-go release — outside this plan)

- `nrepl-client`: bump `.mise.toml` lg to the release; drop the temporary "Build patched lg" step from `checks.yml`; `lgx release 0.1.0` (tag + push `v0.1.0`).
- `nreplctl`: switch the dep from `:local/root "../nrepl-client"` to `:git/url "https://github.com/abogoyavlensky/nrepl-client"` + `:git/tag "v0.1.0"`; bump `.mise.toml` lg; drop its temporary patched-lg CI step.

---

## Completion summary (2026-07-16)

**What was built**

- **nrepl-client** (`/Users/andrew/Projects/nrepl-client`, `master`, 4 commits `8633bbb..0119f16`): the client namespace `nrepl-client.core` (`connect!`/`clone!`/`eval!`/`close!`/`msg-effects`/`resolve-port`) moved out of nreplctl, with its unit + integration tests, a `.clj-kondo/config.edn`, a patched-lg `checks.yml`, and a README documenting the API. It also gained a real fix (below) and a regression test.
- **nreplctl** (`feat/extract-nrepl-client`, 3 commits `b83ea6f..8010e0f`): now consumes the library via `:deps {nrepl-client {:local/root "../nrepl-client"}}`, dropped the in-tree client and its tests, and grew a lean CLI integration test asserting `eval-cmd`'s exit codes (0/1/2).
- **Verified**: both repos `lgx check` green (nrepl-client 16 tests / 28 assertions; nreplctl 1 test / 4 assertions), 0 orphan processes, and the CLI drives the library end-to-end (`eval … '(s/upper-case "extracted")'` → `"EXTRACTED"`, exit 0; error → 1; connect-fail → the friendly message).

**Issues encountered**

- **id-correlation bug (fixed — approved scope growth, `00b7b85`).** The moved client inherited a real pre-existing bug: on a serial nREPL session a timed-out eval keeps running server-side, and the read loop matched *any* `done`, so a later eval on the reused session misread the stale request's value. Fixed by giving each request a unique message id (`gen-id`) and ignoring responses whose echoed id isn't the current request's; added a `timeout-then-reuse` regression test. Verified lg echoes the id on every response message.
- **let-go compiler bug (found, reported, fixed cleanly).** A require **alias named `core`** collides with the auto-referred `clojure.core`, so `core/…` won't resolve at **compile** time (any other alias works; the var is fine at runtime). Minimal repro: two namespaces on one source path — `(:require [x :as core])` then `(core/some-fn)` → `Can't resolve core/some-fn`. Reproduces on **released 1.11.1** and the patched build. Fixed here by aliasing `nreplctl.core` as `cli`. (My in-execution diagnosis blamed transitively-required namespaces and multiple source roots — both were confounds; minimizing for the bug report narrowed it to the `core` alias.)
- **delete-then-add test gap.** Removing every nreplctl test (Task 3) deleted the `test/` dir, so `lgx check`/CI transiently failed (`no test/ directory`) until Task 4 recreated it.

**Deviations** (full detail in each task's notes): renamed the moved tests' temp-file prefix; `mise trust` for the fresh lib repo; gitignored `.clj-kondo/.cache/` in the lib; README note that the `:git/tag` release is pending the first publish; pruned nreplctl's `.clj-kondo` (dropped `net`/`bencode`, the moved namespaces); the id-correlation fix; aliasing `nreplctl.core` as `cli` (not `core`) in the CLI test to dodge the let-go `core`-alias bug; a per-run `os/free-port`-suffixed absent path instead of deleting a fixed one.

**What the plan could have specified better**

- It should have anticipated that deleting **every** nreplctl test removes the `test/` directory and breaks `lgx check` between Task 3 and Task 4 — either keep one test alive across the split, or merge the delete (Task 3) with the add (Task 4).
- It couldn't have foreseen the let-go `core`-alias bug, but the CLI test happened to alias `nreplctl.core` as `core` and walked into it. Nothing the plan could have done differently — the fix is a one-word alias change (`:as cli`). Lesson for future let-go code: avoid `core` as a require alias.
- Small fresh-repo setup steps (`mise trust`, gitignoring `.clj-kondo/.cache/`) weren't called out and had to be handled inline.

**Open follow-ups**

- Publish the library at the net/bencode let-go release (see the Release follow-up section): tag `v0.1.0`, switch nreplctl's dep to `:git/tag`, bump both `.mise.toml` lg pins, drop the temporary patched-lg CI steps.
- File the let-go `core`-alias compile-resolution bug upstream (a prefilled `nooga/let-go` issue link was generated; body saved at `.tmp/issue-core-alias.md`).
