# truffleruby feedstock

TruffleRuby (GraalVM Community, native mode) as a tebako **runtime** —
the spec 28 §8 implementation axis's `truffleruby` flavor of the `ruby`
engine (`ruby:truffleruby`): the release ships the pair per platform,
the tebako-owned wrapper exe (`tebako-runtime-launcher`, the process
entry point) plus the env image (`.tfs`, mounted — never extracted).

- **kind:** runtime (`engine: ruby`, `implementation: truffleruby`)
- **upstream:** TruffleRuby Community 34.0.1 (oracle/truffleruby,
  graal-34.0.1) — the native-mode archives, repacked (no compilation)
- **artifacts:** `tebako-runtime-<tebako-line>-truffleruby-34.0.1-<platform>`
  (the tebako#716 language segment is the flavor's implementation) +
  `.tfs` + `.sha256` sidecars + release shards, the derived
  `manifest.json` index + `SHA256SUMS.txt`, and this registry
  (`tpkg-registry.yaml`) on the repo's releases
- **platforms:** aarch64-macos, x86_64-linux-gnu, aarch64-linux-gnu —
  upstream ships NO windows and NO macos-x86_64 build; the registry
  declares the matrix declaratively
- **visibility:** `exec-cache` (spec 29 §3) with the link-unit preload
  shim granted (the jail survives the exec). The native launcher
  resolves its home exe-relative and dies materialized alone
  ("Loading libjli failed") — preload (tier 1) can never work for it;
  the home tree materializes whole into the spec 22 §6 exec cache.

Consumers' app payloads declare
`runtime_requirement: {engine: ruby, implementation: truffleruby, constraint: "~> 34"}`
on their entrypoints (or select `ruby:truffleruby` through the spec 28
grammar); the dispatcher resolves the newest compatible cached runtime
(or downloads + verifies it from this repo's release index).

The **jvm-mode sibling** (truffleruby on `java:graalvm`, spec 33's
runtime-on-runtime composition) ships from this
same recipe's second flavor: the env image is the truffleruby JVM home,
composed ON the published graalvm java owner at dispatch (the
`on_runtime` edge — graalvm ONLY; temurin fails by name). The mode is a
NON-axis for selection (spec 28 §8): both modes answer
`ruby:truffleruby`; the jvm mode is pinned per-entry
(`truffleruby-jvm`) or by the `;tebako=` line (native rides 2.4.0, jvm
rides 2.5.0 — the spec-33-aware launcher). Early signal (2026-09-07,
fib34 kernel): the native mode runs at 0.148× MRI (~6.8× faster); the
jvm mode as composed (CE owner, `-XX:-UseJVMCINativeLibrary`) runs
interpreted.

The pair's artifact names carry the language segment
([tebako#716](https://github.com/tamatebako/tebako/issues/716)): new
publishes spell `tebako-runtime-<tebako-line>-truffleruby-<version>-<platform>`
(+ `.tfs` for that triplet's env image), where `truffleruby` is this
runtime's distribution identity — an implementation of the ruby engine,
carried by BOTH modes (native and jvm; the mode stays a non-axis and
keeps riding the tebako line). Releases already published keep the
segment-less spelling forever: they are immutable and sha256-pinned in
this registry, and re-running an old tag composes that ref's own
names, self-consistently. The jvm flavor's fetched **owner** pair
(`java:graalvm`) follows the pinned owner release's own era —
`Tebakofile`'s `owner_smoke` block gains an `implementation: graalvm`
key when its pin moves to a release whose assets carry the segment.

## The macOS dyld plane (tebako#1117)

The hardened runtime strips `DYLD_INSERT_LIBRARIES` at exec on macOS 14
when the EXEC TARGET's own signature lacks
`com.apple.security.cs.allow-dyld-environment-variables` (the
tebako-runtime-r#5 class). Under exec-cache neither flavor has an
insertion-dependent core IO:

- The **wrapper** (tebako-signed): its boot-head self-insert fires on
  macOS, but the wrapper only mounts, materializes, and execs —
  nothing in its process consumes the interpose dylib (the same
  product wrapper runs green on darwin-23 in the openjdk line with no
  entitlements at all).
- The **native launcher** (GraalVM CE's bytes, byte-identical): the
  home tree materializes whole (the launcher resolves home
  exe-relative — preload can never serve it), so core IO is host-real
  and shim-free. The `preload_shim` grant's insertion (the jail
  binding) is also never at risk of stripping here: upstream GraalVM
  CE ships the launcher ADHOC-signed (`flags=0x20002
  (adhoc,linker-signed)` — verified on the published v2.8.49 pair's
  materialized home), and dyld strips the variable only for
  hardened-runtime targets. The published v2.8.49 pair boots green on
  darwin-23 (macOS 14.1.1 arm64): `truffle-boot-ok truffleruby 3.4.9`
  with the home-relative read served.
- The **jvm-mode sibling** composes on the graalvm owner exactly the
  jruby way (spec 33 §2's bridge law: the JVM's home materializes
  whole; the owner's signature governs any insertion into it).

The wrapper's profile (`tools/runtime.entitlements`, the pair)
therefore needs no dyld grant; the question reopens only if a shape
change makes a tebako-signed exe the insertion target — this repack
feedstock never builds one.
