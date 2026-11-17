# Phase 0: Validation and decisions (weeks 0–8)

Tracker for Phase 0 of [ROADMAP.md](../ROADMAP.md). Phase 0 ends when every exit criterion below is met.

## Exit criteria

- [ ] Fork-or-upstream decision signed off ([ADR 0001](../decisions/0001-fork-or-upstream.md))
- [x] Name cleared (EUIPO, DPMA): findings in [name-clearance.md](name-clearance.md), "MUJO" signed off 2026-11-17 (#3)
- [ ] CI builds `mujo-mq` and `mujo-chat` reproducibly, including the Android APK
- [ ] Spike results documented in [spikes/](../spikes/)
- [ ] Both developers have landed a change in each repo
- [ ] Month estimates redone for two developers

## Tasks

### 1. Strategic gate (owner: both)
- [ ] Contact SimpleX Chat Ltd; questions in [ADR 0001](../decisions/0001-fork-or-upstream.md#questions-for-simplex-chat-ltd)
- [ ] Record their answers in the ADR and decide

### 2. Name and domain (owner: both)
- [x] Register `mujo.chat` (Cloudflare, ~35.20 USD/yr as of 2026-11-17)
- [x] EUIPO search: "MUJO", classes 9 and 38
- [x] DPMA search: "MUJO", classes 9 and 38
- [x] Assess known conflicts: US mark (Mujo LLC, class 9), app "MUJO: Simple Bullet Journal"
- [x] Shortlist 2 fallback names with free `.chat` or `.app` domains (results: [name-clearance.md](name-clearance.md))

### 3. Build baseline (owner: both)
- [x] `mujo-mq`: Nix dev shell (`flake.nix`, GHC 9.6.3 from nixos-23.11)
- [x] `mujo-mq`: local build with tests at `-O0`
- [x] `mujo-mq`: full test suite run, results recorded in [baseline.md](baseline.md) (980 examples, no deterministic failures)
- [x] `mujo-chat`: CLI build (`simplex-chat` exe) against upstream simplexmq pin
- [x] `mujo-chat`: test suite run, results recorded in [baseline.md](baseline.md) (1979 examples; "Save query plans" still open)
- [ ] `mujo-chat`: Android APK build on CI (#6; needs haskell.nix cross-build + Android SDK). Exit criterion: Phase 1 needs a branded Android build
- [ ] CI on `mujo-org` GitHub: reduced build green on both repos
- [ ] Redo the month estimates in [ROADMAP.md](../ROADMAP.md) for two developers

### 4. Spikes (owner: both)
- [ ] [S1 Replication](../spikes/README.md#s1-two-relay-replication): code analysis done ([results](../spikes/s1-replication.md)), prototype next
- [ ] [S2 Key hierarchy](../spikes/README.md#s2-hardware-key-hierarchy-timing)
- [ ] [S3 Cover traffic](../spikes/README.md#s3-cover-traffic-cost)

### 5. Team skills (owner: both; no hiring planned)
- [ ] Ramp-up: each developer lands one small fix per repo

### 6. Governance and admin
- [ ] Contributor licence policy (AGPL-3.0 inherited, own code stays open)
- [x] Draft [upstream sync policy](../upstream-sync.md)
- [x] Start [fork-diff](../fork-diff.md) log
- [ ] Funding model shortlist (donations / foundation / paid extras)
