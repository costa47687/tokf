# tokf

[![CI](https://github.com/mpecan/tokf/actions/workflows/ci.yml/badge.svg)](https://github.com/mpecan/tokf/actions/workflows/ci.yml)
[![crates.io](https://img.shields.io/crates/v/tokf)](https://crates.io/crates/tokf)
[![crates.io downloads](https://img.shields.io/crates/d/tokf)](https://crates.io/crates/tokf)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

**[tokf.net](https://tokf.net)** — reduce LLM context consumption from CLI commands by 60–90%.

Commands like `git push`, `cargo test`, and `docker build` produce verbose output packed with progress bars, compile noise, and boilerplate. tokf intercepts that output, applies a TOML filter, and emits only what matters — so your AI agent sees a clean signal instead of hundreds of wasted tokens.

---

## Before / After

**`cargo test` — 61 lines → 1 line:**

<table>
<tr>
<th>Without tokf</th>
<th>With tokf</th>
</tr>
<tr>
<td>

```
   Compiling tokf v0.2.0 (/home/user/tokf)
   Compiling proc-macro2 v1.0.92
   Compiling unicode-ident v1.0.14
   Compiling quote v1.0.38
   Compiling syn v2.0.96
   Compiling serde_derive v1.0.217
   Compiling serde v1.0.217
   ...
running 47 tests
test config::tests::test_load ... ok
test filter::tests::test_skip ... ok
test filter::tests::test_keep ... ok
test filter::tests::test_extract ... ok
...
test result: ok. 47 passed; 0 failed; 0 ignored
  finished in 2.31s
```

</td>
<td>

```
✓ 47 passed (2.31s)
```

</td>
</tr>
</table>

**`git push` — 8 lines → 1 line:**

<table>
<tr>
<th>Without tokf</th>
<th>With tokf</th>
</tr>
<tr>
<td>

```
Enumerating objects: 5, done.
Counting objects: 100% (5/5), done.
Delta compression using up to 10 threads
Compressing objects: 100% (3/3), done.
Writing objects: 100% (3/3), 312 bytes | 312.00 KiB/s, done.
Total 3 (delta 2), reused 0 (delta 0), pack-reused 0
remote: Resolving deltas: 100% (2/2), completed with 2 local objects.
To github.com:user/repo.git
   a1b2c3d..e4f5a6b  main -> main
```

</td>
<td>

```
ok ✓ main
```

</td>
</tr>
</table>

---


## Quickstart

```sh
brew install mpecan/tokf/tokf   # or: cargo install tokf
tokf setup                      # detect your AI tools and install hooks
```

That's it. Every command your AI agent runs is now automatically filtered.

Run `tokf gain` to see how many tokens you've saved, or `tokf setup --refresh` to re-run detection.

---


## Installation

### Homebrew (macOS and Linux)

```sh
brew install mpecan/tokf/tokf
```

### cargo

```sh
cargo install tokf
```

### Build from source

```sh
git clone https://github.com/mpecan/tokf
cd tokf
cargo build --release
# binary at target/release/tokf
```

---

## How it works

```
tokf run git push origin main
```

tokf looks up a filter for `git push`, runs the command, and applies the filter. The filter logic lives in plain TOML files — no recompilation required. Anyone can author, share, or override a filter.

Compressing output is not the only thing being in the middle is good for. A pipeline reports its *last* stage's exit code, so `just check 2>&1 | tail -8` reports `tail`'s success even when the tests failed — and the shortcuts an agent reaches for to keep output small are exactly the ones that discard the failure signal. Opt into [pipeline capture](#pipeline-capture) and tokf runs the first stage itself, so it can tell you when a pipeline hid a command's verdict.

---

## Set up automatic filtering

If you use an AI coding tool, install the hook so every command is filtered automatically — no `tokf run` prefix needed:

```sh
# Claude Code (recommended: --global so it works in every project)
tokf hook install --global

# OpenCode
tokf hook install --tool opencode --global

# OpenAI Codex CLI
tokf hook install --tool codex --global
```

Drop `--global` to install for the current project only. See [Claude Code hook](#claude-code-hook) for details on each tool, the `--path` flag, and optional extras like the filter-authoring skill.

---

## Usage

### Run a command with filtering

```sh
tokf run git push origin main
tokf run cargo test
tokf run docker build .
```

### Apply a filter to a fixture

```sh
tokf apply filters/git/push.toml tests/fixtures/git_push_success.txt --exit-code 0
```

### Verify filter test suites

```sh
tokf verify                    # run all test suites
tokf verify git/push           # run a specific suite
tokf verify --list             # list available suites and case counts
tokf verify --json             # output results as JSON
tokf verify --require-all      # fail if any filter has no test suite
tokf verify --list --require-all  # show coverage per filter
tokf verify --scope project    # only project-local filters (.tokf/filters/)
tokf verify --scope global     # only user-level filters (~/.config/tokf/filters/)
tokf verify --scope stdlib     # only built-in stdlib (filters/ in CWD)
tokf verify --safety           # run safety checks (prompt injection, shell injection, hidden unicode)
tokf verify git/push --safety  # safety check a specific filter
```

### Task runner filtering

tokf automatically wraps `make` and `just` so that each recipe line is individually filtered:

```sh
make check    # each recipe line (cargo test, cargo clippy, ...) is filtered
just test     # same — each recipe runs through tokf
```

See [Rewrite configuration](#rewrite-configuration-rewritestoml) for details and customization.

### Explore available filters

```sh
tokf ls                    # list all filters
tokf which "cargo test"    # which filter would match
tokf show git/push         # print the TOML source
```

### Customize a built-in filter

```sh
tokf eject cargo/build            # copy to .tokf/filters/ (project-local)
tokf eject cargo/build --global   # copy to ~/.config/tokf/filters/ (user-level)
```

This copies the filter TOML and its test suite to your config directory, where it shadows the built-in. Edit the ejected copy freely — tokf's priority system ensures your version is used instead of the original.

### Flags

| Flag | Description |
|---|---|
| `--timing` | Print how long filtering took |
| `--verbose` | Show which filter was matched (also explains skipped rewrites) |
| `--no-filter` | Pass output through without filtering |
| `--no-cache` | Bypass the filter discovery cache |
| `--no-mask-exit-code` | Disable exit-code masking. By default tokf exits 0 and prepends `Error: Exit code N` on failure. Also propagates into hook-emitted `tokf run` rewrites (`tokf hook --no-mask-exit-code handle`), including each segment of compound `&&`/`;`/`\|\|` commands |
| `--preserve-color` | Preserve ANSI color codes in filtered output (env: `TOKF_PRESERVE_COLOR=1`). See [Color passthrough](#color-passthrough) below |
| `--baseline-pipe` | Pipe command for fair baseline accounting (injected by rewrite) |
| `--prefer-less` | Compare filtered vs piped output and use whichever is smaller (requires `--baseline-pipe`) |

### Color passthrough

By default, filters with `strip_ansi = true` permanently remove ANSI escape codes. The `--preserve-color` flag changes this: tokf strips ANSI **internally** for pattern matching (skip, keep, dedup) but restores the original colored lines in the final output. When `--preserve-color` is active it overrides `strip_ansi = true` in the filter config.

tokf does **not** force commands to emit color — you must ensure the child command outputs ANSI codes (e.g. via `FORCE_COLOR=1` or `--color=always`):

```sh
# Node.js / Vitest / Jest
FORCE_COLOR=1 tokf run --preserve-color npm test

# Cargo
tokf run --preserve-color cargo test -- --color=always

# Or set the env var once for all invocations
export TOKF_PRESERVE_COLOR=1
FORCE_COLOR=1 tokf run npm test
```

**Limitations:** color passthrough applies to the skip/keep/dedup pipeline (stages 2–2.5). The `match_output`, `parse`, and `lua_script` stages operate on clean text and are unaffected by this flag. `[[replace]]` rules run on the raw text before the color split, so when `--preserve-color` is enabled their patterns may need to account for ANSI escape codes, similar to branch-level `skip` patterns, which also match against the restored colored text.

---

## Built-in filter library

| Filter | Command |
|---|---|
| `git/add` | `git add` |
| `git/commit` | `git commit` |
| `git/diff` | `git diff` — runs the real `git diff` and summarises it as one line per file (`src/main.rs \| +4 -3`) plus a totals line, so the full patch stays recoverable with `tokf raw`. Pass `-p`/`--patch`/`--stat`/`-U<n>`/`--name-only`/`--name-status`/`--numstat`/`--shortstat`/`--raw` to skip the filter and get the requested format instead |
| `git/log` | `git log` — runs the real `git log` and renders one line per commit (`<short-sha> <subject>`), capped at 20; the full history stays recoverable with `tokf raw`. Pass `-p`/`--patch`/`--format`/`--pretty`/`--graph`/`--stat`/`--shortstat`/`--dirstat`/`--oneline`/`--name-only`/`--name-status`/`-L` to skip the filter. Empty results emit a one-line hint pointing at common causes (untracked pathspec, missing `--all`, missing `--follow`) instead of nothing — this stops agents looping through flag variations trying to escape a non-existent filter |
| `git/push` | `git push` |
| `git/show` | `git show` — runs the real `git show` and renders the commit's sha, subject, author and date plus one line per file with change counts; the full patch stays recoverable with `tokf raw`. Pass `-p`/`--patch`/`--stat`/`--format`/`--pretty`/`--numstat`/`--shortstat`/`--raw` to skip the filter |
| `git/status` | `git status` — runs `git status --porcelain=v1 -b -uall --find-renames`; shows branch + upstream sync state (`[synced]`, `[ahead N]`, `[behind N]`, `(no upstream)`) and one porcelain line per changed file (`M  src/main.rs`, `?? scratch.rs`, `R  old.rs -> new.rs`). `-uall` lists every untracked file individually instead of collapsing newly-created directories. When 3+ files share a directory prefix the listing is restructured into a directory tree (see [`[tree]`](writing-filters.md#tree-restructuring)), writing each shared prefix once. Measured 24.4% averaged token reduction across the bundled test fixtures |
| `cargo/build` | `cargo build` |
| `cargo/check` | `cargo check` |
| `cargo/clippy` | `cargo clippy` |
| `cargo/fmt` | `cargo fmt` |
| `cargo/install` | `cargo install *` |
| `cargo/test` | `cargo test` |
| `docker/*` | `docker build`, `docker compose`, `docker images`, `docker ps` |
| `npm/run` | `npm run *` |
| `npm/test` | `npm test`, `pnpm test`, `yarn test` (with vitest/jest variants) |
| `pnpm/*` | `pnpm add`, `pnpm install` |
| `go/*` | `go build`, `go vet` |
| `gradle/*` | `gradle build`, `gradle test`, `gradle dependencies` |
| `gh/*` | `gh pr list`, `gh pr view`, `gh pr checks`, `gh issue list`, `gh issue view` |
| `kubectl/*` | `kubectl get pods` |
| `next/*` | `next build` |
| `prisma/*` | `prisma generate` |
| `pytest` | Python test runner — runs `pytest` as typed and keeps the failing assertion lines (`>` / `E`) plus the pass/fail summary; the full tracebacks stay recoverable with `tokf raw`. Pass `-q`/`--tb`/`-v`/`-x`/`--collect-only`/`--pdb` to skip the filter |
| `tsc` | TypeScript compiler |
| `ls` | `ls` |

---


## Generic Commands

When no dedicated filter exists for a command, three built-in subcommands provide useful compression for arbitrary output:

| Command | Purpose | Default context |
|---------|---------|----------------|
| `tokf err <cmd>` | Extract errors and warnings | 3 lines |
| `tokf test <cmd>` | Extract test failures | 5 lines |
| `tokf summary <cmd>` | Heuristic summary | 30 lines max |

### `tokf err` — Error extraction

Scans output for error/warning patterns across common toolchains (Rust, Python, Node, Go, Java) and shows only the relevant lines with surrounding context.

```sh
# Show only errors from a build
tokf err cargo build

# Adjust context lines around each error
tokf err -C 5 cargo build

# Works with any command
tokf err python train.py
tokf err npm run build
```

**Patterns matched:** `error:`, `warning:`, `FAILED`, `Traceback`, `panic`, `npm ERR!`, `fatal:`, Python/Java exception types, and more.

**Behaviour:**
- Empty output: `[tokf err] no errors detected (empty output)`
- Short output (< 10 lines): shows `[tokf err]` header with full output
- No errors + exit 0: prints `[tokf err] no errors detected`
- No errors + exit ≠ 0: includes full output (something failed but no recognized pattern)

### `tokf test` — Test failure extraction

Extracts test failure details and always includes summary/result lines.

```sh
# Show only test failures
tokf test cargo test
tokf test go test ./...
tokf test npm test
tokf test pytest
```

**Patterns matched:** `FAIL`, `FAILED`, `panicked`, assertion mismatches, Jest `✕` markers, Go `--- FAIL:`, and more.

**Summary lines always included:** `test result:`, `Tests:`, `passed`, `failed` counts, pytest/RSpec summary lines.

**Behaviour:**
- Output < 10 lines: passed through unchanged
- All pass + exit 0: prints `[tokf test] all tests passed`
- Failures detected: shows failure lines with context + summary

### `tokf summary` — Heuristic summary

Produces a budget-constrained summary by identifying header, footer/summary, and repetitive middle sections.

```sh
# Summarize a long build log
tokf summary cargo build

# Limit to 15 lines
tokf summary --max-lines 15 make all
```

**Algorithm:**
1. Header (first 5 lines) and footer/summary (last lines matching keywords like "total", "finished", "result")
2. Middle section is sampled; highly repetitive content shows a count + samples
3. Extracted statistics (pass/fail counts, timing) appended as `[tokf summary]` line

### Common flags

All three commands support:

| Flag | Description |
|------|-------------|
| `--baseline-pipe <cmd>` | Fair baseline accounting (as with `tokf run`) |
| `--no-mask-exit-code` | Propagate the real exit code instead of masking to 0 |
| `--timing` | Show how long filtering took |

### Using generic commands with rewrites

Generic commands can be integrated with the [rewrite system](rewrites-config.md) so they trigger automatically through the hook. Add rules to `.tokf/rewrites.toml` for commands that don't have dedicated filters:

```toml
# Route build commands without filters through tokf err
[[rewrite]]
match = "^mix compile"
replace = "tokf err {0}"

[[rewrite]]
match = "^cmake --build"
replace = "tokf err {0}"

# Route test runners without filters through tokf test
[[rewrite]]
match = "^mix test"
replace = "tokf test {0}"

[[rewrite]]
match = "^ctest"
replace = "tokf test {0}"

# Summarize long-running commands
[[rewrite]]
match = "^terraform plan"
replace = "tokf summary {0}"
```

**Important:** User rewrite rules are checked *before* filter matching. Don't add rules for commands that already have dedicated filters (like `cargo build`, `npm test`) — the dedicated filter will produce better output than the generic command.

To check whether a command already has a filter: `tokf which "cargo build"`.

### Tracking and history

Generic commands record to the same tracking database and history as `tokf run`, using filter names `_builtin/err`, `_builtin/test`, and `_builtin/summary`. Use `tokf raw last` to see the full uncompressed output.

---


Filters are TOML files placed in `.tokf/filters/` (project-local) or `~/.config/tokf/filters/` (user-level). Project-local filters take priority over user-level, which take priority over the built-in library.

## Minimal example

```toml
command = "my-tool"

[on_success]
output = "ok ✓"

[on_failure]
tail = 10
```

## Command matching

tokf matches commands against filter patterns using two built-in behaviours:

**Basename matching** — the first word of a pattern is compared by basename, so a filter with `command = "git push"` will also match `/usr/bin/git push` or `./git push`.  This works automatically; no special pattern syntax is required.

**Transparent global flags** — flag-like tokens between the command name and a subcommand keyword are skipped during matching.  A filter for `git log` will match all of:

```
git log
git -C /path log
git --no-pager -C /path log --oneline
/usr/bin/git --no-pager -C /path log
```

The skipped flags are preserved in the command that actually runs — they are only bypassed during the pattern match.

> **Note on `run` override and transparent flags:** If a filter sets a `run` field, transparent global flags are *not* included in `{args}`.  Only the arguments that appear after the matched pattern words are available as `{args}`.

**Local environment wrappers** — you don't need to do anything special for your filter to match through a local wrapper like `nix develop -c cargo test`. tokf strips the wrapper prefix and matches the inner command (`cargo test`) against your existing patterns. See [Local environment wrappers](rewrites-config.md#local-environment-wrappers) for the configurable list.

## Common fields

```toml
command = "git push"          # command pattern to match (supports wildcards and arrays)
run = "git push {args}"       # override command to actually execute
description = "Compact git push output"  # human-readable description (shown in `tokf ls`)

skip = ["^Enumerating", "^Counting"]  # drop lines matching these regexes
keep = ["^error"]                      # keep only lines matching (inverse of skip)

# Per-line regex replacement — applied before skip/keep, in order.
# Capture groups use {1}, {2}, … . Invalid patterns are silently skipped.
[[replace]]
pattern = '^(\S+)\s+\S+\s+(\S+)\s+(\S+)'
output = "{1}: {2} → {3}"

dedup = true                  # collapse consecutive identical lines
dedup_window = 10             # optional: compare within a N-line sliding window

strip_ansi = true             # strip ANSI escape sequences before processing
trim_lines = true             # trim leading/trailing whitespace from each line
strip_empty_lines = true      # remove all blank lines from the final output
collapse_empty_lines = true   # collapse consecutive blank lines into one
truncate_lines_at = 120       # truncate lines longer than N chars (with trailing …)

tail = 30                     # keep last N lines regardless of exit code (branch tail overrides)
on_empty = "git push: ok"     # message when filter produces empty output (all lines stripped)

show_history_hint = true      # append a hint line (`tokf raw <id>`) pointing to the full output in history
inject_path = true            # inject shims into PATH so sub-processes (e.g. git hooks) are filtered

passthrough_args = ["--watch", "--web", "-w"]  # skip filter when user passes these flags

# Lua escape hatch — for logic TOML can't express (see Lua Escape Hatch section)
[lua_script]
lang = "luau"
source = 'return output:upper()'    # inline script
# file = "transform.luau"           # or reference a local file (auto-inlined on publish)

match_output = [              # whole-output substring checks, short-circuit the pipeline
  { contains = "rejected", output = "push rejected" },
]

[on_success]                  # branch for exit code 0
output = "ok ✓ {2}"          # template; {output} = pre-filtered output

[on_failure]                  # branch for non-zero exit
tail = 10                     # keep the last N lines (overrides top-level tail)
```

## The `run` override

`run` makes tokf execute a *different* command than the user typed. It is a sharp
tool, and there is one rule:

> **`run` must not lose information.** It may re-encode the same data more densely
> (`--porcelain`, `--format json`, `-o json`). It must never truncate, cap, or
> otherwise answer a narrower question than the user asked.

The reason is recoverability. tokf only ever sees the output of the command it
actually ran, so that is what lands in history and what `tokf raw <id>` gives
back. If `run` throws data away before tokf sees it, **nothing can recover it** —
not `tokf raw`, not anything else. Reductions that drop content belong in the
filter pipeline (`skip`, `chunk`, `max_l