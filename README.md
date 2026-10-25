# Google Search — standalone Rust extension

Independent stdio executable using `tezbar-extension-sdk`; no Tezbar app crate, Tauri, JavaScript runtime, or user-machine Cargo build is needed for a packaged installation. Source manifest identity is `com.tezbar.native.google-search`, schema 3, protocol 1.

Two commands preserve the installed behavior: Google autosuggestions with its Latin-1 response decoding, history ordering/deduplication, current URL encoding, selected-text search, and clipboard fallback only after selection fails and the preference is enabled. Selected-text searches retain the existing history-write behavior, including when `rememberSearchHistory` is false. Empty-string selection behavior is retained rather than silently redesigned.

HTTP requests have an eight-second client deadline and a one-MiB response cap. SDK latest-search cancellation drops the in-flight request and rejects stale state commits. History uses scoped SDK storage; legacy Raycast history is read only from a host-granted `legacyLocalStorage` path. Tests use local HTTP fixtures and captured host selection/clipboard/open/hide RPCs; no real Google request is needed.

## Build and test

Cargo.toml pins the standalone SDK to immutable commit `9f91afcc071c3f301414dc62b33cfeac7d89a0cf`. The app source is not a build input. Keep `Cargo.lock` and use one crate at a time:

```sh
cargo test --locked -j1
cargo build --locked -j1 --bin tezbar-google-search
./target/debug/tezbar-google-search --protocol-info
./target/debug/tezbar-google-search --self-test
```

The executable without flags speaks UTF-8 LF JSON-lines on stdin/stdout; logs go to stderr. It emits `hello` first and waits for `initialize` containing protocol/identity, absolute scoped package/data/temp directories, granted generic capabilities and explicit migration paths. Sessions use `session.open`, `session.call` with the original `extension:*` operation names, and `session.dispose`. Call IDs are `h:<UUID>` and `e:<UUID>`; reverse host responses must be serviced while an operation is pending. `shutdown` or stdin EOF triggers lifecycle cleanup. `--protocol-info` and `--self-test` do not access hardware, launch apps, or contact providers.

Integration tests run the real executable through pipes for protocol-info, hello, initialize, open, dispose and EOF. The Google/CleanShot tests also answer real reverse RPCs; Speedtest uses a separate fixture process for subprocess behavior. No tests are ignored to obtain a passing run.

## Scoped data and migration

Storage is `dataDir/google-search/storage.json`, using SDK atomic transactional replacement and corruption rejection. Existing private keys win over imported native keys; unknown keys are retained. The host must resolve these **manifest-declared relative legacy sources** into explicit trusted absolute `initialize.migrationPaths` entries:

- `nativeStore`: `native-extensions-v2/google-search/storage.json`
- `legacyLocalStorage`: `extensions/google-search/.tezbar-local-storage.json`

Only named grants are read. No app-global config, installed extension directory discovery or credential copying occurs. Migration sources are never modified. `packageDir` is the validated immutable package root; `dataDir` and `tempDir` are extension-private.

## Package releases

`extension.json.artifacts` is empty in source; published catalog metadata supplies verified target downloads. To update the SDK pin, review the immutable revision and run:

```sh
python3 scripts/pin-sdk.py --rev 9f91afcc071c3f301414dc62b33cfeac7d89a0cf
cargo test --locked -j1
```

The script rejects placeholders/non-SHA revisions. Inspect and commit the final dependency and lockfile as part of parent publication. Release packaging and CI reject a local/unpinned SDK dependency. The native CI matrix uses GitHub-hosted `macos-15` ARM64 and `windows-2022` x64 MSVC runners. Each job tests, builds, packages and uploads review artifacts, with no release publishing step.

Build with `--target aarch64-apple-darwin` or `--target x86_64-pc-windows-msvc`, then use:

```sh
python3 scripts/package.py --target aarch64-apple-darwin --binary target/aarch64-apple-darwin/release/tezbar-google-search
```

For a review-only local development ZIP, add `--allow-local-sdk`. `python3 scripts/smoke-package.py dist/google-search-aarch64-apple-darwin.zip` validates its full inventory and runs the extracted executable offline over real stdio without installing it. CI runs the same check. The ZIP has `extension.json`, `bin/tezbar-google-search[.exe]`, `resources/**`, licenses, LICENSE and NOTICE at its root. The embedded `files` map inventories every packaged file except the manifest. Sidecars contain the real ZIP SHA-256 and executable SHA-256; parent supplies the actual release URL and pins those values in the trusted catalog after release. Never publish development ZIPs or invent artifact URLs/digests. End-user installation executes the verified packaged binary; it never runs Cargo or a helper provisioning script.

## Command coverage

| Original command ID | Implemented platform coverage |
|---|---|
| `index` | macOS, Windows |
| `searchSelectText` | macOS, Windows |

macOS ARM unit/process tests are recorded in the task handoff. Windows code/CI is present but has not been natively executed by this implementation task. Tests do not establish full hardware/provider/host-install acceptance.

## Licensing and provenance

Keep `LICENSE` (original Tezbar Apache-2.0), `licenses/raycast-extensions-MIT.txt`, `NOTICE`, original artwork and author/contributor metadata. `provenance/source.json` describes the input snapshot, not the port's validation status; `provenance/upstream` distinguishes installed metadata from later upstream context. The exact installed upstream commit is unknown. Original icons/resources were copied unchanged; no new trademark rights or universal asset license are asserted. Separate helper/provider notices apply where present.
