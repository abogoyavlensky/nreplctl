# nreplctl

A basic let-go CLI application template.

## Development

Install dependencies with [mise](https://mise.jdx.dev/getting-started.html) (or manaully consulting the `.mise.toml` file):

```bash
mise trust && mise install
```

Run main application commands during development:
```bash
lgx --help
lgx run
lgx run -- help
lgx run -- greet
lgx test
lgx fmt
lgx lint
lgx nrepl
```

Buld a binary and use it:

```bash
lgx build
bin/nreplctl --help
bin/nreplctl --version
bin/nreplctl help greet
bin/nreplctl greet
```
