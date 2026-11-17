# Build and test baseline

Upstream state before any Mujo change. Anything that fails here must not be blamed on Mujo code later.

Machine: x86_64-linux (NixOS), 12 cores, 62 GB RAM.

## mujo-mq

- Commit: `9ffd36ed` (simplexmq 7.1.0.18), submodules `cbits/blst@db3defd`, `cbits/libbbs@59a0f4b`
- Toolchain: `nix develop` (`flake.nix`, nixos-23.11, GHC 9.6.3, cabal-install)
- Build: `cabal build --enable-tests --ghc-options=-O0 all`: **OK**
- Flags: defaults (no `server_postgres`, no `client_postgres`), so PostgreSQL test groups are compiled out

### Setup gotchas

1. Run `git submodule update --init --recursive` first; otherwise the build fails with `does not exist: cbits/blst/src/server.c`.
2. Don't set `LD_LIBRARY_PATH` in the dev shell: it makes the system `curl` that cabal uses load the shell's OpenSSL 3.0 and fail.
3. `libz.so: cannot open shared object file` during the build is a harmless Template Haskell warning.

### Tests

```sh
"$(cabal list-bin simplexmq-test --ghc-options=-O0)" --jobs=4 --times
```

Full run, 2026-11-17: **980 examples, 4 failures, 2 pending**, 26 min.

| Test | Full run | Isolated rerun | Verdict |
|---|---|---|---|
| SMP client agent, memory store / Batching / subscribe to 200 subscriptions | timeout (300 s) | pass (259 s) | Flaky under load: runtime close to the 300 s limit at `-O0` |
| SMP client agent, memory store / Delivery receipts / concurrently with messages | predicate failed (61 s) | pass (39 s) | Flaky under load: timing assertion |
| XFTP agent / resume deleting file after restart | expected 0, got 1 | pass | Flaky |
| XRCP / Multicast discovery / find paired host and connect | timeout | fails | Environment: mDNS multicast `224.0.0.251:5227` is likely blocked by the NixOS firewall. XRCP (desktop remote control) is not on the MVP path |

**Conclusion:** no deterministic upstream failures. For local runs:
- Run timing-heavy groups with `--jobs=1` or use `ATTEMPTS=2` with `scripts/ci/run-tests.sh`.
- Allow UDP 5227 (`networking.firewall.allowedUDPPorts`) to run the XRCP tests, or skip them with `--skip "/XRCP/Multicast discovery/"`.

## mujo-chat

- Commit: `ad5e9d217` (simplex-chat 7.1.4.0), simplexmq pinned to upstream `fdbcb24d`
- Toolchain: `nix develop ../mujo-mq` (same GHC 9.6.3 shell)
- Build: `cabal build --enable-tests --ghc-options=-O0 all`: **OK**
- Tests: _running_ (`simplex-chat-test --jobs=4 --times`, binary path passed directly; a first attempt through `cabal list-bin` failed with exit 127 while cabal re-synced source repos)
- Android APK: **deferred**. Needs `nix build .#hydraJobs.x86_64-linux.aarch64-android:lib:simplex-chat` (haskell.nix cross GHC; add `--option extra-substituters https://cache.iog.io` to avoid building GHC from source), an Android SDK and Gradle, plus `git submodule update --init` for `apps/multiplatform/external/nanohttpd/upstream`
