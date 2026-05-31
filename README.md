# homebrew-verus

A [Homebrew](https://brew.sh) tap that installs [Verus](https://github.com/verus-lang/verus) —
the tool for verifying the correctness of code written in Rust.

> **Repo name matters.** Homebrew taps must live in a repo named `homebrew-<name>`. Push
> this to `github.com/<you>/homebrew-verus` so that `brew tap <you>/verus` resolves.

## Install

```sh
brew install <you>/verus/verus
```

(or `brew tap <you>/verus` then `brew install verus`)

### One required follow-up: the Rust toolchain

Verus runs against a **specific rustup-managed Rust toolchain** that is *not* bundled and
*not* installed by Homebrew (it can be hundreds of MB and may be shared with your other
Rust projects). The first time you run `verus`, it prints the exact command to install it:

```text
verus: required rust toolchain 1.95.0-<your-target-triple> not found
run the following command (in a bash-compatible shell) to install the necessary toolchain:
  rustup install 1.95.0-<your-target-triple>
```

Run that `rustup install …` command once and `verus` works. The toolchain version is
deliberately **not** baked into the formula — Verus self-reports the one it needs, so the
formula never has to change when upstream bumps the compiler.

## What gets installed

The release archive is a directory — the `verus` launcher plus `cargo-verus`, the `z3`
solver, the `vstd` standard library, and supporting files. `verus` locates those relative
to its own path, so the formula installs the whole tree into `libexec` and exposes only the
two launchers via wrapper scripts:

| Command      | Exposed in `bin`? | Notes                                                 |
| ------------ | ----------------- | ----------------------------------------------------- |
| `verus`      | yes               | the verifier                                          |
| `cargo verus`| yes (`cargo-verus`) | cargo subcommand integration                        |
| `z3`         | **no**            | kept in `libexec` to avoid clashing with the `z3` formula |
| `rust_verify`| **no**            | internal backend                                      |

## Platform support

| Platform              | Asset                          | Supported |
| --------------------- | ------------------------------ | --------- |
| macOS (Apple Silicon) | `verus-<ver>-arm64-macos.zip`  | ✅        |
| macOS (Intel)         | `verus-<ver>-x86-macos.zip`    | ✅        |
| Linux (x86_64)        | `verus-<ver>-x86-linux.zip`    | ✅        |
| Linux (arm64)         | —                              | ❌ upstream ships no arm64-linux build |
| Windows               | `verus-<ver>-x86-win.zip`      | ❌ Homebrew is macOS/Linux only |

Tracks the **stable** weekly releases (`release/<date>.<hash>`). The rolling pre-release
channel (`release/rolling/…`) is intentionally not followed.

## Uninstall

```sh
brew uninstall verus
```

removes everything Homebrew installed (launchers, Z3, vstd, the wrappers) — it all lives in
the keg. `brew cleanup` handles cached downloads and old versions as usual.

**Left in place on purpose** (not bugs):

- The rustup-managed toolchain in `~/.rustup/toolchains/`. It may be shared with other Rust
  projects, so removing it automatically would be the actual mistake. Reclaim it yourself with:
  ```sh
  rustup toolchain uninstall <version>-<your-target-triple>
  ```
- `rustup` itself survives `brew uninstall verus`. `brew autoremove` clears it only if
  Homebrew installed it solely as a dependency of Verus. `~/.rustup` and `~/.cargo` persist
  regardless.
- Runtime files Verus writes (logs, `--record` archives, caches) are outside Homebrew's
  tracking, as with essentially every CLI tool.

## Maintenance model — generated, not hand-edited

`Formula/verus.rb` is **generated output**. Do not edit it by hand. All formula logic lives
once in [`scripts/generate-formula.sh`](scripts/generate-formula.sh); the only thing that
changes per release is data (the version and three per-platform `sha256`s), and that data is
produced by machine from the exact release artifacts.

Regenerate locally for any release tag:

```sh
scripts/generate-formula.sh release/0.2026.05.24.ecee80a > Formula/verus.rb
# or hash already-downloaded artifacts instead of fetching them:
scripts/generate-formula.sh release/0.2026.05.24.ecee80a /path/to/zips > Formula/verus.rb
```

Two ways to keep it current, in order of preference:

1. **Release-integrated (best, zero lag).** Upstream's release workflow regenerates and
   pushes the formula as its final step — brew support becomes a free side-effect of every
   release. The ~15-line job to propose is in
   [`docs/upstream-publish-step.yml`](docs/upstream-publish-step.yml); the pitch is in
   [`docs/UPSTREAM_PROPOSAL.md`](docs/UPSTREAM_PROPOSAL.md).
2. **Scheduled fallback (this repo).** [`.github/workflows/bump-formula.yml`](.github/workflows/bump-formula.yml)
   checks daily for a newer stable release and opens a bump PR. A PR opened by the default
   `GITHUB_TOKEN` does **not** trigger `tests.yml`, so kick CI on the bump branch manually
   (or give the workflow a fine-grained PAT so CI runs, then enable auto-merge). Used only if
   upstream declines to own step 1.

The committed formula is generator output. CI
([`.github/workflows/tests.yml`](.github/workflows/tests.yml)) is configured to run a real
install + test + audit across arm64 macOS, Intel macOS, and x86_64 Linux on every push and
PR; to date that cycle has been validated by hand on arm64 macOS only (see Validation status).

## Validation status

Currently pinned: **`0.2026.05.24.ecee80a`**. Verified locally on arm64 macOS with Homebrew
5.1.14: `brew style`, `brew audit --strict --online`, `brew audit --new`, and a real
`brew install` → `brew test` → `brew upgrade` (from the prior stable) → `brew uninstall`,
all clean.

## License

The tap (this repo) is [MIT](LICENSE). Verus itself is MIT, © the Verus authors.
