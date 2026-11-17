# Fork diff log

Every deliberate difference from upstream, newest first. One row per change.

| Date | Repo | Change | Upstream base | Why |
|---|---|---|---|---|
| 2026-11-17 | mujo-mq | Added `flake.nix` dev shell (GHC 9.6.3, nixos-23.11) | `9ffd36ed` | Reproducible local builds on NixOS |
| 2026-11-17 | mujo-chat | Added upstream derivative notice (`ad5e9d217`) | `cb4b50db1` | AGPL attribution |

## Known divergences still to make (Phase 1)

- `mujo-chat/cabal.project` pins `simplex-chat/simplexmq@fdbcb24d`. Switch it to `mujo-org/mujo-mq` once that has its first Mujo-specific commit.
- App IDs, names and preset servers.
