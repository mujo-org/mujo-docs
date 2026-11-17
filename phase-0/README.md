# Phase 0: Validation and decisions (weeks 0–8)

Tracker for Phase 0 of [ROADMAP.md](../ROADMAP.md). Phase 0 ends when every exit criterion below is met.

## Exit criteria

- [ ] Fork-or-upstream decision signed off ([ADR 0001](../decisions/0001-fork-or-upstream.md))
- [ ] Name cleared (EUIPO, DPMA) or replaced: findings in [name-clearance.md](name-clearance.md), sign-off pending
- [ ] CI builds `mujo-mq` and `mujo-chat` reproducibly
- [ ] Spike results documented in [spikes/](../spikes/)
- [ ] Experienced Haskell engineer on board

## Tasks

### 1. Strategic gate (owner: founders)
- [ ] Contact SimpleX Chat Ltd; questions in [ADR 0001](../decisions/0001-fork-or-upstream.md#questions-for-simplex-chat-ltd)
- [ ] Record their answers in the ADR and decide

### 2. Name and domain (owner: founders)
- [ ] Register `mujo.chat` (Cloudflare, ~35.20 USD/yr as of 2026-11-17)
- [x] EUIPO search: "MUJO", classes 9 and 38
- [x] DPMA search: "MUJO", classes 9 and 38
- [x] Assess known conflicts: US mark (Mujo LLC, class 9), app "MUJO: Simple Bullet Journal"
- [x] Shortlist 2 fallback names with free `.chat` or `.app` domains (results: [name-clearance.md](name-clearance.md))

### 3. Build baseline (owner: core devs)
- [x] `mujo-mq`: Nix dev shell (`flake.nix`, GHC 9.6.3 from nixos-23.11)
- [x] `mujo-mq`: local build with tests at `-O0`
- [x] `mujo-mq`: full test suite run, results recorded in [baseline.md](baseline.md) (980 examples, no deterministic failures)
- [x] `mujo-chat`: CLI build (`simplex-chat` exe) against upstream simplexmq pin
- [ ] `mujo-chat`: test suite run, results recorded in [baseline.md](baseline.md)
- [ ] `mujo-chat`: Android APK build (deferred: needs haskell.nix cross-build + Android SDK)
- [ ] CI on `mujo-org` GitHub: reduced build green on both repos

### 4. Spikes (owner: core devs, Android dev)
- [ ] [S1 Replication](../spikes/README.md#s1-two-relay-replication): code analysis done ([results](../spikes/s1-replication.md)), prototype next
- [ ] [S2 Key hierarchy](../spikes/README.md#s2-hardware-key-hierarchy-timing)
- [ ] [S3 Cover traffic](../spikes/README.md#s3-cover-traffic-cost)

### 5. Team (owner: founders)
- [ ] Job post: senior Haskell engineer (SimpleX codebase experience is a plus)
- [ ] Ramp-up: each developer lands one small fix per repo

### 6. Governance and admin
- [ ] Contributor licence policy (AGPL-3.0 inherited, own code stays open)
- [x] Draft [upstream sync policy](../upstream-sync.md)
- [x] Start [fork-diff](../fork-diff.md) log
- [ ] Funding model shortlist (donations / foundation / paid extras)
