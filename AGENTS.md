# AGENTS.md

Guidance for AI coding agents (GitHub Copilot, Claude Code, Cursor, Aider,
Codex, and similar) working in the `embedded-fans` repository. This file
complements `.github/copilot-instructions.md` and `CONTRIBUTING.md` — when
guidance differs, the more specific document wins. Human contributors are
welcome to read this too, but the audience is autonomous and assistive
agents.

If you are an agent reading this for the first time on a new task, read
this entire file before making changes. It is intentionally short.

---

## 1. Repository at a glance

- **Name:** `embedded-fans`
- **Owner:** [`openDevicePartnership`](https://github.com/openDevicePartnership/embedded-fans)
- **Purpose:** A Hardware Abstraction Layer (HAL) for fans used in
  embedded systems. The crates define generic `Fan` and `RpmSense`
  traits plus a common `Error` / `ErrorKind` / `ErrorType` hierarchy
  that HAL implementations consume. There is no concrete hardware
  driver in this repository — only the abstract traits.
- **Language / edition:** Rust, edition `2021`.
- **MSRV:** `1.79` (declared in each crate's `Cargo.toml` and enforced
  by the `msrv` CI job).
- **Runtime:** `no_std`. Both crates are `#![no_std]` and CI verifies
  this on `thumbv7m-none-eabi` and `aarch64-unknown-none`.
- **Safety posture:** `#![forbid(unsafe_code)]` and
  `#![forbid(missing_docs)]` in every crate. Do not introduce `unsafe`
  or undocumented public items.
- **License:** MIT.
- **Default branch:** `main`.

### Workspace layout

This is a Cargo workspace with two members. Both are published to
crates.io.

```
.
├── Cargo.toml                     # workspace root (resolver = "2")
├── embedded-fans/                 # blocking trait crate
│   ├── Cargo.toml
│   ├── README.md
│   └── src/lib.rs
├── embedded-fans-async/           # async trait crate (depends on embedded-fans)
│   ├── Cargo.toml
│   ├── README.md
│   └── src/lib.rs
├── .github/
│   ├── copilot-instructions.md    # AI agent commit-message rules
│   ├── DOCS.md                    # notes about CI provenance
│   ├── codecov.yml
│   ├── dependabot.yml
│   └── workflows/
│       ├── check.yml              # fmt, clippy, semver, doc, hack, msrv
│       └── nostd.yml              # no_std target builds
├── CONTRIBUTING.md
├── CODEOWNERS
├── CODE_OF_CONDUCT.md
├── SECURITY.md
└── README.md
```

### Workspace members

| Crate | Path | Role | Notable attrs |
| --- | --- | --- | --- |
| `embedded-fans` | `embedded-fans/` | Blocking `Fan` / `RpmSense` traits, `Error`, `ErrorKind`, `ErrorType`. | `#![no_std]`, `#![forbid(unsafe_code)]`, `#![forbid(missing_docs)]` |
| `embedded-fans-async` | `embedded-fans-async/` | Async `Fan` / `RpmSense` traits. Re-exports `Error`, `ErrorKind`, `ErrorType` from `embedded-fans`. | Same as above plus `#![allow(async_fn_in_trait)]` |

The workspace `Cargo.toml` contains a `[patch.crates-io]` entry so
that `embedded-fans-async` always resolves the local `embedded-fans`
when built from the workspace, even though its declared dependency is
versioned. Preserve this patch when editing the root manifest.

### Features

Each crate exposes a single optional feature:

- `defmt` — derives `defmt::Format` on public error types and pulls in
  `defmt` 1.x as an optional dependency. In `embedded-fans-async`,
  enabling `defmt` also enables `embedded-fans/defmt` (feature
  unification).

`cargo hack --feature-powerset` runs in CI; any new feature must be
purely additive.

---

## 2. Public API contract

The public surface is intentionally tiny. Agents must treat it as a
**stable, semver-checked API**: `cargo-semver-checks` runs on every
PR (`check.yml` → `semver` job) and a breaking change requires a
major-version bump in the affected crate's `Cargo.toml`.

### Blocking crate (`embedded-fans`)

- `trait Error: core::fmt::Debug` with `fn kind(&self) -> ErrorKind`.
- `enum ErrorKind` — `Peripheral`, `InvalidSpeed`, `Other`. Marked
  `#[non_exhaustive]`; agents may add variants without a major bump,
  but must update the `Display` impl and document the new variant.
- `trait ErrorType { type Error: Error; }` with a blanket
  `impl<T: ErrorType + ?Sized> ErrorType for &mut T`.
- `trait Fan: ErrorType` with required methods `max_rpm`, `min_rpm`,
  `min_start_rpm`, `set_speed_rpm`, plus provided methods
  `set_speed_percent`, `set_speed_max`, `start`, `stop`. A blanket
  `impl<T: Fan + ?Sized> Fan for &mut T` mirrors every method.
- `trait RpmSense: ErrorType` with `fn rpm(&mut self) -> Result<u16, Self::Error>`,
  plus a `&mut T` blanket impl.

### Async crate (`embedded-fans-async`)

- Re-exports `Error`, `ErrorKind`, `ErrorType` from `embedded-fans`
  (`pub use embedded_fans::{Error, ErrorKind, ErrorType};`). **Do not
  duplicate** these types — they are intentionally shared so a single
  error implementation works for both APIs.
- `trait Fan: ErrorType` — same shape as blocking, but
  `set_speed_rpm`, `set_speed_percent`, `set_speed_max`, `start`, and
  `stop` are `async fn`. A `&mut T` blanket impl is provided.
- `trait RpmSense: ErrorType` with `async fn rpm`.

### Invariants to preserve

- Keep the two trait surfaces **structurally identical** apart from
  `async` markers. If you add a method to one crate, add it to the
  other in the same PR with the same name, parameters, and docs.
- Default-method implementations must perform identical arithmetic in
  both crates. The percent-scaling formula
  `(u32::from(max_rpm) * u32::from(percent)) / 100` is intentional —
  it avoids `u16` overflow. Preserve the `debug_assert!((0..=100).contains(&percent))`.
- `impl Error for core::convert::Infallible` lives in the blocking
  crate and is reused via the async crate's re-export. Do not add a
  second impl.
- Every public item must have a doc comment (enforced by
  `#![forbid(missing_docs)]`). Examples in the module-level `//!`
  doc must compile — they are run as doctests.

---

## 3. Local toolchain & commands

Agents should prefer the same commands CI runs so that local
verification matches the gate. All commands run from the workspace
root unless noted.

### Toolchain

- Stable Rust with components `rustfmt`, `clippy`.
- Nightly Rust for `cargo doc` (only required if you want to mirror
  the CI doc job; see below).
- The pinned MSRV is `1.79`; `cargo check --locked` must succeed on
  that exact toolchain.
- For `no_std` verification you also need the targets
  `thumbv7m-none-eabi` and `aarch64-unknown-none`.

```sh
rustup component add rustfmt clippy
rustup target add thumbv7m-none-eabi aarch64-unknown-none
```

### Required pre-PR checks

Run these and fix any failure before handing work back to a human:

```sh
cargo fmt --all -- --check
cargo clippy --all-targets --all-features --locked -- -D warnings
cargo test --all-features --locked
cargo check --locked
cargo check --target thumbv7m-none-eabi --no-default-features --locked
cargo check --target aarch64-unknown-none --no-default-features --locked
```

### Nice-to-have checks (CI also runs these)

```sh
# Feature powerset — needs cargo-hack
cargo install cargo-hack --locked
cargo hack --feature-powerset check --locked

# Semver compatibility — needs cargo-semver-checks
cargo install cargo-semver-checks --locked
cargo semver-checks

# Docs — nightly only
cargo +nightly doc --no-deps --all-features --locked
```

If `cargo install` is unavailable in your sandbox, document the skip
in your final report rather than silently bypassing the check.

### CI workflows summary

- `.github/workflows/check.yml` — jobs `fmt`, `clippy` (stable + beta),
  `semver`, `doc` (nightly, with `RUSTDOCFLAGS=--cfg docsrs`), `hack`,
  `msrv` (1.79, `cargo check --locked`).
- `.github/workflows/nostd.yml` — `cargo check --target <t> --no-default-features --locked`
  on `thumbv7m-none-eabi` and `aarch64-unknown-none`.

All jobs run on `push` to `main` and on every `pull_request`.

---

## 4. Coding conventions

### Style

- `rustfmt` is the single source of truth. Do not hand-format.
- Clippy must be clean with `-D warnings`. If a lint is genuinely
  wrong for the context, prefer a narrowly scoped `#[allow(...)]` on
  the item with a `// reason:` comment over a workspace-wide allow.
- Keep `#![no_std]` at the top of every `lib.rs`. Do not add `std`
  imports. `alloc` is not currently used; introduce it only if
  necessary and gate it behind a feature.
- Do not add new `unsafe` code. `#![forbid(unsafe_code)]` is in
  effect; agents must not weaken it.
- Every public item needs a `///` doc comment that explains the
  semantics (not just the signature). Mention units (RPM, percent),
  panicking behaviour, and error conditions where relevant.

### Trait additions

When extending a trait:

1. Add the method to **both** `embedded-fans::Fan` and
   `embedded-fans-async::Fan` (or both `RpmSense` traits) with
   matching signatures.
2. Provide a default implementation when feasible so downstream HALs
   are not broken — recall that adding a required method without a
   default is a semver-breaking change.
3. Update the `&mut T` blanket impl in both crates.
4. Update the module-level doctest in both `lib.rs` files to exercise
   the new method.

### Error handling

- Return `Result<_, Self::Error>` from fallible methods. Never panic
  in trait default methods on input from a caller; `debug_assert!` is
  acceptable for contract violations such as `percent > 100`.
- New `ErrorKind` variants must:
  - be added to `ErrorKind` in `embedded-fans/src/lib.rs`,
  - have a `Display` arm,
  - be documented,
  - be considered for `defmt::Format` automatically (the derive
    handles new variants without code changes).

### Dependencies

- New runtime dependencies require justification. Keep the dependency
  footprint minimal — currently the only runtime dep is optional
  `defmt`.
- Pin to a permissive but explicit version (e.g., `1.0.1` matching
  existing style). Run `cargo hack` after adding a feature to verify
  the powerset still builds.
- Dependabot is configured in `.github/dependabot.yml`; do not disable
  it.

---

## 5. Commit, branch, and PR workflow

### Branches

- Default branch: `main`. Do not push directly to `main`.
- Work on a topic branch in your fork. Branch names should be short
  and descriptive (`kebab-case`).
- The project disables squash-merging. **Every commit on your branch
  must build cleanly and pass CI.** Reorganise (`git rebase -i`)
  before pushing the final version.

### Commit messages

From `.github/copilot-instructions.md`:

- Subject line: capitalised, ≤ 50 characters, imperative mood
  (`"Add foo"`, not `"Added foo"` or `"Adds foo"`).
- Blank line between subject and body.
- Wrap body at 72 characters.
- Body explains **what** and **why**, not **how**.
- Squash fix-up commits (typo / formatting fixes) before pushing.

### AI attribution (mandatory)

Every commit that contains AI-generated or AI-assisted work **must**
include an `Assisted-by` trailer:

```
Assisted-by: AGENT_NAME:MODEL_VERSION [TOOL1] [TOOL2]
```

- `AGENT_NAME` — the AI tool/framework, e.g. `GitHub Copilot`,
  `Claude Code`, `Cursor`.
- `MODEL_VERSION` — the exact model identifier, e.g.
  `claude-opus-4.7`, `gpt-5.2`. **Verify your own identity at the
  start of every session.** Do not copy a model name from a prior
  session, a sibling repository, or this file's examples.
- Optional trailing tokens list specialised analysers (`coccinelle`,
  `sparse`, `smatch`, `clang-tidy`, ...). Do not list everyday tools
  like `git`, `cargo`, editors, or general-purpose LLM chat.
- AI agents **must not** add `Signed-off-by`. Only humans can certify
  the Developer Certificate of Origin.

Example:

```
Add tachometer averaging helper

Provide a default `rpm_averaged` method on `RpmSense` that
returns the mean of N consecutive samples. Helpful for HALs
whose sensors are noisy at low speeds.

Assisted-by: GitHub Copilot:claude-opus-4.7
```

### Pull requests

- Open a **draft PR first** so CI runs early.
- Confirm `fmt`, `clippy`, `semver`, `doc`, `hack`, `msrv`, and the
  two `nostd` jobs are green before requesting review.
- Keep the PR scoped to a single logical change. Mechanical refactors
  go in their own PR.
- Add a brief PR description explaining motivation and any semver
  implications.

### Things to never do

- Force-push to `main` or to other contributors' branches.
- Rewrite history that has already been merged.
- Add `Signed-off-by` from an AI account.
- Commit secrets, tokens, `.env` files, or anything from a local
  `target/` directory.
- Bypass MSRV by using a `1.80+` API.
- Remove `#![forbid(unsafe_code)]` or `#![forbid(missing_docs)]`.
- Introduce divergence between the blocking and async trait surfaces.

---

## 6. Working effectively as an agent

### Before editing

1. Read this file and `.github/copilot-instructions.md`.
2. Read both `src/lib.rs` files end-to-end — the entire library fits
   on two screens; you have no excuse for surprise.
3. Check whether a similar method already exists with a different
   name before adding a new one.

### While editing

- Mirror every public-API change across the blocking and async
  crates.
- Update doctests, not just code. The module-level `//!` examples are
  contracts — they appear in published rustdoc.
- Keep changes minimal and surgical. Do not opportunistically
  reformat unrelated code; `rustfmt` already handles formatting.

### Before committing

Run the full required-checks list from §3. If any of `fmt`,
`clippy`, `test`, `check`, or the two `nostd` `check` targets fail,
fix the failure or stop and report it — never disable the lint or
skip the target.

### When unsure

Prefer the smaller change, the more explicit name, and the safer
default. If a design decision is ambiguous (for example, whether a
new method should default-implement in terms of `set_speed_rpm` or
require implementors to override it), leave a comment in the PR
description rather than guessing silently.

---

## 7. Quick reference

| Need | Command |
| --- | --- |
| Format check | `cargo fmt --all -- --check` |
| Lint | `cargo clippy --all-targets --all-features --locked -- -D warnings` |
| Test | `cargo test --all-features --locked` |
| MSRV check | `cargo +1.79 check --locked` |
| no_std build | `cargo check --target thumbv7m-none-eabi --no-default-features --locked` |
| no_std build | `cargo check --target aarch64-unknown-none --no-default-features --locked` |
| Feature powerset | `cargo hack --feature-powerset check --locked` |
| Semver | `cargo semver-checks` |
| Docs (nightly) | `RUSTDOCFLAGS='--cfg docsrs' cargo +nightly doc --no-deps --all-features --locked` |

| File | Purpose |
| --- | --- |
| `Cargo.toml` | Workspace root + `[patch.crates-io]` for local `embedded-fans` |
| `embedded-fans/src/lib.rs` | Blocking traits + error types |
| `embedded-fans-async/src/lib.rs` | Async traits, re-exports error types |
| `.github/workflows/check.yml` | Main CI matrix (fmt, clippy, semver, doc, hack, msrv) |
| `.github/workflows/nostd.yml` | `no_std` target builds |
| `.github/copilot-instructions.md` | Commit-message and AI-attribution rules |
| `CONTRIBUTING.md` | Licensing, PR etiquette, clean-history policy |

---

## 8. Pilot-specific notes

The rules above apply to every agent. The notes in this section call
out tool-specific nuances. If your tool is not listed, follow the
GitHub Copilot section as the baseline.

### GitHub Copilot (Chat, CLI, Coding Agent)

- Treat `.github/copilot-instructions.md` as authoritative for
  commit-message formatting and the `Assisted-by` trailer. This file
  re-states the rules for convenience but does not override them.
- When the Coding Agent runs `cargo` commands, prefer `--locked` so
  the lockfile is respected; CI uses `--locked` everywhere.
- The Coding Agent should open a **draft** PR and wait for CI before
  requesting review.

### Claude Code / Claude in IDE

- Verify the model identifier (e.g. `claude-opus-4.7`,
  `claude-sonnet-4.6`) at session start and use it verbatim in the
  `Assisted-by` trailer. Do not abbreviate.
- Claude tends to expand scope — keep edits surgical and respect the
  workspace structure described in §1.

### Cursor / Windsurf / other IDE agents

- Honour `rustfmt` on save; do not introduce formatting drift.
- Disable any "auto-apply unsafe suggestion" behaviour; this repo
  forbids `unsafe`.

### Aider, Codex, and other CLI agents

- Use a topic branch and run the §3 commands before each commit.
- When the agent generates multiple commits, ensure each commit
  builds standalone (`cargo check --locked` per commit) to satisfy
  the clean-history rule from `CONTRIBUTING.md`.

### Dependabot

- Dependabot is not an authoring agent, but its PRs follow the same
  CI gates. If you (a human or another agent) rebase a Dependabot PR,
  preserve the original commit message and add your own
  `Assisted-by` trailer only if you materially changed the code.

---

End of `AGENTS.md`. Keep this file current — when CI, MSRV,
workspace layout, or commit policy changes, update this file in the
same PR.
