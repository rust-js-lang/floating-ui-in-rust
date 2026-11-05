# Floating UI, in Rust

This repository is [floating-ui/floating-ui](https://github.com/floating-ui/floating-ui)
being ported to Rust, module by module, with
[rust-js](https://github.com/rust-js-lang/rust-js): each TypeScript module
is replaced by a Rust module that rust-js compiles to the JavaScript, and
the `.d.ts`, beside it, which reads as the original does. Floating UI's own
builds and tests run on that JavaScript as they ran on the original.

The tree started as floating-ui/floating-ui at
[`e1e9f88`](https://github.com/floating-ui/floating-ui/commit/e1e9f88993c19426563decfcc991feb706f1ac66)
(2026-11-04). This repository's root commit, `11cbd8e`, is that tree, byte
for byte. Then it builds with Bun workspaces instead of pnpm and Turborepo,
without upstream's GitHub Actions and changesets, and with a crate for each
of `utils`, `core` and `dom`. What isn't Rust yet is still upstream's
TypeScript.

What Rust to write, and the JS rust-js makes of it, is in
[rust-js's guide](../rust-js/docs/guide/README.md): read it first. rust-js's
[AGENTS.md](../rust-js/AGENTS.md) is the north star both repositories
follow.

## Setting up

Clone the two repositories side by side, and install [Bun](https://bun.sh)
and [rustup](https://rustup.rs):

```sh
git clone git@github.com:rust-js-lang/rust-js.git
git clone git@github.com:rust-js-lang/floating-ui-in-rust.git
(cd rust-js && bun install)
cd floating-ui-in-rust && bun install
```

floating-ui-in-rust reaches rust-js only through `../rust-js`:

- **Cargo:** the workspace's `Cargo.toml` takes `js` and `webapi` as path
  dependencies, `../rust-js/builtins` and `../rust-js/webapi`.
- **npm:** `@rust-js/runtime` and `@rust-js/std`, which the generated JS
  imports, are `link:`s to `../rust-js/runtime` and `../rust-js/std`.
- **The compiler:** `bun run rust` is `../rust-js/tooling/cli.js build`,
  which builds the compiler from that checkout when its sources changed.

Pull rust-js and the port uses it: there is nothing to reinstall.

## Building

Each ported package is a crate in the workspace, and the crate's modules
are the package's files: `packages/core/src/computePosition.rs` compiles to
`computePosition.js` and `computePosition.d.ts` beside it. A crate's root is
Cargo.toml's `[lib] path`, `packages/utils/utils.rs`, `packages/core/core.rs`
or `packages/dom/dom.rs`. It declares each ported module by its file,
`#[path = "src/computePosition.rs"] pub mod compute_position;`, and writes
no JS of its own. The crates depend on each other as their packages do:
`core` on `utils`, `dom` on both. A package ported later is a new crate,
added to the workspace's `members`.

```sh
bun run rust                                        # every crate to its JS, at once
bun run rust --package floating-ui-core             # one crate, and what it uses
cargo fmt -p floating-ui-core                       # rustfmt, this crate only
node ../rust-js/tooling/cli.js clippy -- -D warnings
```

- **The generated `.js` and `.d.ts` are committed**, as upstream's
  TypeScript was. Source maps are not (`.gitignore`).
- **Run `cargo fmt -p <crate>`, never `cargo fmt --all`**: `--all` follows
  the path dependencies into rust-js's crates.
- **Use rust-js's clippy** (its ADR 0888): it must be clean with
  `-D warnings`. A plain `cargo clippy` says what only JS reads is dead.
- **Your editor** checks the crates through rust-js:
  `.vscode/settings.json` sets rust-analyzer's check to
  `../rust-js/tooling/check.js` (its ADR 0222).

## The gates

Before every commit, run the gates, from the root. Their results must
match these, measured on the tree at the port's setup; a change in a count
is a finding to explain, not noise.

| Gate | Command | Result |
|---|---|---|
| Build | `bun run build` | the 9 workspaces build: `config`, then `utils`, `core`, `dom`, `react-native`, `vue`, `devtools`, `react-dom`, `react` |
| Unit tests | `bun run test` | 424 passed, 18 skipped: utils 24; core 30; dom 34, 12 skipped; react-dom 18; vue 32; react 286, 6 skipped |
| Types | `bun run typecheck` | every workspace passes |

`bun run --filter '*' <task>` runs each workspace's task after its
dependencies' tasks, as Turborepo's `^build` did. `bun run --filter
@floating-ui/core test` runs one package's.

`bun run playwright` runs `dom`'s functional tests against 1017
screenshots of Linux's Chromium, `*-linux.png`. Playwright names a
snapshot by its platform, so they run on Linux: a Mac has none to compare
with.

## Porting a module

Port a package's modules leaves first, so each module's imports are
already settled: `utils`, then `core`, then `dom`; then `react-dom`, `vue`
and `react`, which build on them.

1. Write `src/<name>.rs` beside `src/<name>.ts`, as an experienced Rust
   engineer would write that module, with std's types and idioms. Declare
   it in the crate's root by its path.
2. `bun run rust`, then delete `src/<name>.ts`: `src/<name>.js` and
   `src/<name>.d.ts` take its place, and the rest of the package imports
   them as it imported it.
3. Compare the generated JS with the original line by line, and the
   `.d.ts` with the original's types. Every difference is fixed in
   rust-js, unless the user accepts it by name.
4. Run the gates. Commit the `.rs`, the `.js` and the `.d.ts` together.

**Stop the port and fix rust-js** when it refuses valid Rust, lacks a
binding, writes JS that differs from the original (a rename, a helper, a
copy, a lost name), or when the Rust must be bent to fit it: rust-js's
AGENTS.md says how. Never hide a gap with a local stand-in binding or
a workaround in the source.

The first ported module of a package will meet the package's Rollup build,
whose `nodeResolve` looks only for `.ts` and `.tsx`
(`config/src/defineRollupConfig.mts`): it has to find the generated `.js`
too.
