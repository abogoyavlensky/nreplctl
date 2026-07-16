# nreplctl

Evaluate Clojure code in a running [nREPL](https://nrepl.org) server from the
command line and print the results. A small [let-go](https://github.com/nooga/let-go)
CLI built to be pleasant for humans and predictable for agents: friendly
failures, meaningful exit codes, and REPL-faithful output.

It speaks the standard nREPL ops (`clone` / `eval` / `close`) over
bencode-on-TCP, so it works against let-go's own nREPL server, a JVM `nrepl`
server, and babashka.

## Usage

```
nreplctl eval '<code>' [-p/--port N] [--port-file PATH] [--host HOST] [-t/--timeout SECONDS]
```

```bash
nreplctl eval '(+ 1 1)'                              # → 2
nreplctl eval --port 2137 '(println "hi")'          # → hi
nreplctl eval '(require (quote [clojure.string :as s])) (s/join ", " ["a" "b" "c"])'
```

### Options

| Flag | Default | Meaning |
|---|---|---|
| `<code>` | *(required)* | Clojure code to evaluate — one positional. Multiple forms in one string are sent as-is; the server evaluates them all. |
| `-p`, `--port N` | — | nREPL port. Overrides `--port-file`. |
| `--port-file PATH` | `.nrepl-port` | File to read the port from when `--port` is absent. |
| `--host HOST` | `127.0.0.1` | nREPL host. |
| `-t`, `--timeout SECONDS` | *(off)* | Bound each read during eval. On expiry: `Eval timed out after Ns` to stderr, exit 1. |

### Port discovery

1. `--port` if given (must be a positive integer); it wins over any file.
2. otherwise the port in `--port-file` (default `.nrepl-port` in the current directory).
3. otherwise a friendly error to stderr and exit 2.

### Output

All diagnostics go to **stderr**; **stdout** carries only evaluated output and
values. Every evaluated value prints on its own line — every value, not just
the last — `out` output streams as it arrives, and `err` output goes to stderr.

Per-form value reporting depends on the server: babashka sends one value
message per form, while let-go's own nREPL server reports only the last form's
value.

```bash
# against babashka (one value per form: require → nil, then the join result)
$ nreplctl eval '(require (quote [clojure.string :as s])) (s/join ", " ["a" "b" "c"])'
nil
"a, b, c"
```

### Exit codes

nreplctl is built to be scripted and driven by agents.

| Code | Meaning | Example stderr message |
|---|---|---|
| `0` | eval succeeded | |
| `1` | code threw, or eval timed out | the server's `err` output / `Eval timed out after 30s` |
| `2` | couldn't resolve the port, connect, or complete the protocol | `No .nrepl-port file found and no --port given — is an nREPL server running here?` · `Could not connect to nREPL at 127.0.0.1:2137 — is the server still running?` |

## Install

Install dependencies with [mise](https://mise.jdx.dev/getting-started.html) (or
consult `.mise.toml`):

```bash
mise trust && mise install
```

Build a binary:

```bash
lgx build
bin/nreplctl --help
bin/nreplctl eval '(+ 1 1)'
```

## Development

```bash
lgx run -- eval '(+ 1 1)'   # run from source
lgx test                    # run the test suite
lgx fmt                     # format
lgx lint                    # lint (clj-kondo)
lgx check                   # fmt check + lint + test
```

> **Temporary — building lg from source.** nreplctl needs let-go's `net` and
> `bencode` namespaces, which are not yet in a released `lg`. Until a release
> ships, build lg from the
> [`tcp-client`](https://github.com/abogoyavlensky/let-go/tree/tcp-client)
> branch and point lgx at it via `LGX_LG`:
>
> ```bash
> git clone -b tcp-client https://github.com/abogoyavlensky/let-go /tmp/let-go
> (cd /tmp/let-go && go build -o lg .)
> export LGX_LG=/tmp/let-go/lg
> ```
>
> `.mise.toml` still pins `lg 1.11.1`; once a release ships `net`/`bencode`,
> bump it, drop this note, and remove the "Build patched lg" step from CI.
