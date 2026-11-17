# Mujo Development Roadmap

Derived from *Mujo: Konzept für einen Messenger ohne Nummer und Username* (Konzept, 17.11.2026) and *Mujo White Paper v0.3* (WP). Where the two disagree, **WP v0.3 wins**: it is the later document and explicitly replaces the Konzept's 6–9 month estimate, Android+Desktop MVP, per-mailbox push tokens, shared entry hop and default-on read receipts. Section references (§) point to the white paper.

Team: **2 developers, no hiring planned**. The WP's 12–18 months for an Android MVP assumes 3–4 developers plus a part-time designer, so expect it to take longer with two. Re-estimate once spike S1 shows how much agent code changes, and keep the MVP scope tight.

---

## 0. Starting point

| Repo | Today | Role in Mujo |
|---|---|---|
| `mujo-mq` | Unmodified fork of `simplexmq` 7.1.0.18 (SMP server, agent, XFTP, ntf-server, xftp-web) | Protocol core: replication, circuits, directory client, wake-up tickets, node binary |
| `mujo-chat` | Fork of `simplex-chat` 7.1.4.0 (Haskell core, Kotlin multiplatform Android/desktop, iOS) | Client: read-once storage, key hierarchy, UI redesign, branding |
| `mujo-docs` | Empty | White paper, protocol specs, threat model, this roadmap |
| `mujo-infra` | Empty | Nix/NixOS configs for nodes, directory signing, push services |
| `mujo-web` | Empty | Static landing page for `https://mujo.chat/i#…` invitation links |

What SimpleX already provides and Mujo keeps: identifier-free contacts via one-time links, unidirectional queues, Double Ratchet with PQ KEM (sntrup761), 16 KB padding, XFTP file relays, private message routing through a forwarding relay (a starting point for 2-hop circuits).

What has to be built (WP §11): multi-relay replication with dedup, separate-circuit onion routing with cover traffic, a signed node directory, the wake-up ticket push service, the hardware-backed key hierarchy with read-once storage, and a redesigned UI.

---

## Phase 0 — Validation and decisions (weeks 0–8)

The WP conclusion asks for a 2-month prototype before committing. Its purpose is to decide **fork vs. upstream contribution** and to remove the unknowns that would change the plan.

1. **Strategic gate: fork or contribute upstream?** (WP §12 "Differentiation from SimpleX")
   - Talk to SimpleX Chat Ltd about replication, circuits and multi-device. Find out whether they would accept these upstream, and whether they would grant an App Store licence exception.
   - Outcome: written decision. If upstream is viable, this roadmap becomes a contribution plan.
2. **Name and domain**
   - Register `mujo.chat` (Cloudflare, ~35 USD/yr).
   - Check EUIPO and DPMA for "MUJO" (class 9/38). Known conflicts: US mark held by Mujo LLC, the app "MUJO: Simple Bullet Journal". Pick a fallback name before branding work starts.
3. **Build baseline**
   - Reproducible Nix builds of `mujo-mq` (smp-server, xftp-server) and `mujo-chat` (CLI and Android APK) on CI.
   - Run the upstream test suites green and record the baseline.
4. **Technical spikes** (throwaway code, measured results go into `mujo-docs/spikes/`)
   - **Replication spike:** client deposits one message on 2 SMP servers and the receiver drops the duplicate. Measure the changes needed in `smp-agent`.
   - **Key-hierarchy spike:** on 3 Android devices (one with StrongBox, one TEE-only, one low-end), time EEK create/wrap/delete. Target: under 300 ms tap-to-display (§7.1).
   - **Cover-traffic spike:** measure the battery and data cost of the Table 5 polling schedule on real devices.
5. **Team skills** (no hiring; WP §11 calls Haskell a critical-path skill)
   - Both developers build Haskell competence in the existing codebase: each lands one small fix per repo, and S1 is done in pairs where possible.
   - Keep new protocol code small and well specified, and prefer upstream contributions where SimpleX accepts them (see ADR 0001).
   - Ramp-up plan: each developer lands one small upstream-style fix in each repo.
6. **Governance and admin**
   - Write the contributor licence policy (AGPL inherited; own code stays open).
   - Draft upstream-sync policy: track upstream tags, rebase cadence (e.g. monthly), fork-diff document.
   - Start the funding decision (donations / foundation / paid extras). It doesn't block development, but it blocks operator recruiting.

**Exit criteria:** fork decision signed off; name cleared or replaced; CI builds both repos; spike numbers documented; both developers have landed a change in each repo.

---

## Phase 1 — Fork, rebrand, specs (months 1–3)

WP §11.2 step 1. Konzept phase 1 (months 1–2) is folded in here.

### 1.1 Fork hygiene
- Rename packages, binaries and app IDs (`chat.simplex.app` → own ID). Keep upstream module structure so rebases stay cheap.
- Strip or disable SimpleX-specific services that Mujo doesn't ship (directory/broadcast/support bots, badge service), behind build flags rather than deletions where possible.
- Replace preset servers with Mujo servers. Remove SimpleX operator presets.
- Keep `NOTICE.md` and derivative notices accurate.

### 1.2 Branding
- App name, icon, colours, store listing drafts (designer).
- `mujo-web`: placeholder landing page on `mujo.chat`.

### 1.3 First infrastructure (`mujo-infra`)
- NixOS modules for smp-server and xftp-server, with secrets management and monitoring (metrics without per-mailbox data).
- Deploy relays at **2 operators** (own + 1 partner).
- Logging policy: no IP logging, documented.

### 1.4 Written specifications (`mujo-docs/specs/`)
These must exist before Phase 2/3 code (WP §11: "each needing a written specification before implementation").
1. `replication.md`: 2-relay mailboxes, deposit/fetch rules, dedup by inner-packet hash *before* ratchet advance, relay replacement and re-announcement.
2. `circuits.md`: separate Hop-1 circuit per relay for deposits, separate Hop-2 circuit per relay for fetches, operator-disjointness rules, 10-minute rebuild, 0–10 s random inter-deposit delay, onion layer format.
3. `directory.md`: node descriptor format, operator identity, signing keys, threshold signing, update/rotation, client validation.
4. `key-hierarchy.md`: device root key, EEK, table key, per-message key, rotate-and-rewrap transaction, crash recovery (two EEKs on start-up).
5. `read-once.md`: message lifecycle (§5.1), rules table (§5.2), retention timers, per-chat timer mode.
6. `wake-tickets.md`: Privacy Pass / RFC 9578 blind-RSA issuance, ticket blob format, part A/part B split, refill during fetch.
7. `cover-traffic.md`: Table 5 schedule, send slots, dummy cells, receipt batching (§5.4).
8. `threat-model.md`: Tables 4, 6, 10 kept as the security contract the code must uphold.

**Exit criteria:** branded Android build talks only to Mujo relays at 2 operators; specs 1–5 reviewed internally; specs 6–8 drafted.

---

## Phase 2 — Replication, read-once, key hierarchy (months 4–7)

WP §11.2 step 2. Konzept phase 2 (months 3–5).

### 2.1 Multi-relay mailboxes (`mujo-mq/smp-agent`)
1. Extend the queue model: one logical receive queue = 2 SMP queues on relays of different operators.
2. Invitation links carry both relay addresses (§4.1). Update link format and parser.
3. Sender deposits to both queues; per-relay encryption layer so packets differ on the wire (Table 4, Relay A + Relay B row).
4. Receiver subscribes to both and dedups by inner-packet hash before the ratchet sees it (§6.1, §7.2 rule 4).
5. Relay failure: detect, create replacement queue on a new operator, send new address to contact encrypted (reuse SimpleX queue rotation where possible).
6. Unfetched message expiry on relays: 7 days (server config).
7. Tests: one relay down, one relay slow, reordering, duplicate floods, replay.

### 2.2 Ratchet changes (§7.2)
1. Decrypt at arrival and erase the ratchet message key in the same step.
2. Skipped keys: MAX_SKIP = 1000 per chain, 2000 per contact. Commit state only after authentication.
3. Skipped keys stored in the key table and burned on use; expire after 8 days.
4. Property tests for out-of-order delivery across two relays.

### 2.3 Hardware-backed key hierarchy (Android, `mujo-chat`)
1. Device root key in StrongBox, with TEE Keystore fallback. Non-exportable, excluded from backups.
2. Erasure epoch key (EEK): exactly one live; rollback-resistant where KeyMint supports it.
3. Table key in RAM while unlocked; key table holds per-message keys, ratchet state and skipped keys.
4. Rotate-and-rewrap in one DB transaction; batch reads within a short window into one rotation.
5. Crash recovery: on start, find two EEKs → finish deletion before showing anything.
6. Benchmark against the 300 ms target on the Phase 0 device set.

### 2.4 Read-once storage and display
1. On arrival: re-encrypt content under a fresh per-message key. UI shows "1 new message" only.
2. On open: destroy key (rotate-and-rewrap), decrypt into RAM, display.
3. On leave chat / screen lock / background: wipe plaintext from RAM.
4. Unread retention: 30 days default, configurable.
5. Sender keeps no copy; status only "sent" / "delivered".
6. Disable quote and forward.
7. Per-chat timer mode (e.g. 24 h) as the opt-out from read-once.

### 2.5 Files (XFTP)
- Read-once rules apply to images, voice and files. File relays delete after download, at the latest after 48 h.
- Chunks spread across file relays of different operators.

### 2.6 Small groups (MVP scope)
- Each member reads once; the relay deletes once all members have fetched (§5.2).
- Decide whether the MVP keeps SimpleX fan-out or moves to Sender Keys over pairwise mailboxes (Konzept/WP §8.1, up to 50). **Recommendation:** keep fan-out for the MVP and cap groups at 50. Sender Keys come after the MVP.

**Exit criteria:** two-relay delivery survives one relay outage in an automated test; read-once flow passes crash-injection tests (kill app at every step, nothing readable afterwards); key hierarchy meets or documents the 300 ms target.

---

## Phase 3 — Circuits, directory, cover traffic (months 7–11)

WP §11.2 step 3. Konzept phase 3 (months 5–7).

### 3.1 Node software
- One binary/package for all roles: hop, mailbox relay, file relay (§6.1). Role configured, not compiled.
- NixOS module in `mujo-infra` for each role combination.

### 3.2 Signed node directory
1. Directory format and signing per `directory.md`. Each node is listed with its operator.
2. Directory publishing service and client fetch/validation, with pinned signing keys in the app.
3. Operator vetting process written down. Directory governance is a security property (§10.3, §12).

### 3.3 Separate-circuit routing
1. Onion layer per hop (build on the SMP private-routing proxy).
2. Separate circuit per relay for each deposit and each fetch; operator-disjoint selection (§6.1, Table 4).
3. Circuit rebuild at least every 10 minutes.
4. Independent random 0–10 s delay between the two deposits.
5. When fewer than 6 operators are available, show it in the client instead of hiding it (§12).
6. Optional Tor transport as an alternative to Mujo hops (Advanced settings).

### 3.4 Fixed-rate fetching and cover traffic (§6.4, Table 5)
1. Fetch schedule independent of expected messages, per circuit, with ±20 % jitter.
2. Send slots with dummy cells; everything padded to 16 KB.
3. Profiles: chat open Wi-Fi / mobile, background Wi-Fi / mobile, battery saver. Full cover on Wi-Fi by default, 15-minute polling on mobile data.
4. Show the privacy trade-off on mobile data in the app; opt-in full cover.
5. Read receipts: off by default, opt-in per contact, random 5–60 min delay, batched, sent only in send slots. Delivered acks batched too (§5.4).
6. Measure data and battery on real devices and tune Table 5 values.

### 3.5 Operators (non-dev, starts in Phase 1)
- Recruit partner operators. Target **at least 6 independent operators** for the full guarantee.
- Operator agreement: no logging, incident handling, legal-request handling.

**Exit criteria:** end-to-end messages flow over 2 hops × 2 relays across ≥ 4 operators in staging; directory signed and validated by clients; cover-traffic numbers measured and published in `mujo-docs`.

---

## Phase 4 — Push, UI redesign, usability (months 10–14)

WP §11.2 step 4. Konzept phase 4 (push part).

### 4.1 Wake-up ticket service (§6.5)
1. Part A: blind-RSA token issuance (RFC 9578) over a Hop connection; PoW rate-limiting; token spend check.
2. Part B: push gateway; decrypts ticket blob, sends an empty push through FCM. Part B runs at a **different operator** from part A.
3. Relay side: stores tickets as opaque bytes, sends one to part A per arriving message.
4. Client: obtains token batches, builds tickets with fresh nonces, refills relays during normal fetches.
5. Android without Google: persistent connection mode as an alternative.
6. Retire the upstream ntf-server path (no per-mailbox device tokens anywhere).

### 4.2 UI redesign (designer + Android dev, ~2 months)
1. Only two contact words: **Invite** and **My link** (§9.1).
2. First launch (§9.2): name + optional picture → big Invite button → system share sheet or QR.
3. Plain-language status and errors ("Anna hasn't opened your invitation yet", "Will be delivered when Anna is online").
4. Read-once UX: "Tap to read", first-time hint, flame/smoke icon.
5. Everything technical under Advanced; defaults work.
6. Accessibility: screen reader, large text, high contrast.

### 4.3 Invitation links (`mujo-web`)
1. Link format `https://mujo.chat/i#…`, with all secrets in the fragment.
2. Static landing page, no trackers, reads the fragment only in the browser; store link; **Copy invitation** button; "Open in Mujo" for in-app browsers.
3. Host `assetlinks.json` (and later `apple-app-site-association`).
4. App: Android App Links; **Paste invitation** button; never reads the clipboard on its own.
5. Invitations expire after first use or 24 h.

### 4.4 Contacts and identity features
1. Optional long-term contact address (§4.2): leads only to requests, can be regenerated.
2. Request inbox with accept/decline; proof-of-work per request (~1–2 s on a phone); relay rate limits.
3. Per-contact incognito profiles; local renaming.
4. Verification: QR scan or NFC tap marks a contact verified (green shield); 60-digit safety number as fallback.

### 4.5 Side-copy hardening (§5.3)
- `FLAG_SECURE` (screenshot block, no app-switcher preview), incognito keyboard flag on inputs, notifications without content, app data excluded from Android backups.

### 4.6 Backup (§8.4)
- Encrypted export of contacts and keys only, protected by a 12-word phrase. User picks where to store it.
- Prompt once after the third contact.
- After restore, rotate all ratchets.

### 4.7 Usability testing (§9.4)
- 5 non-technical testers per round, three tasks: set up, invite, read.
- Pass bar: all tasks done without help, first message in under 60 s. At least two rounds before the beta.

**Exit criteria:** push wake-ups work end-to-end on Android without any per-mailbox token stored anywhere; usability bar met in one full round.

---

## Phase 5 — Audit, beta, Android MVP release (months 14–18)

WP §11.2 step 5. Konzept phase 4 (beta/audit part).

1. **Audit booking:** contact auditors by month 8–10, since they book months ahead. Budget EUR 30,000–80,000.
2. **Design freeze** of new protocol parts (replication, circuits, directory, tickets, key hierarchy, read-once).
3. **External audit** of the new protocol and implementation. Publish the report.
4. **Fix cycle** and re-test of audit findings.
5. **Closed beta** with 50–100 testers (F-Droid/APK + Play internal track). Collect crash, delivery and battery data without content or identities.
6. **Release readiness:**
   - Security claims in the store listing and the app match the platform erasure table (§7.1, §12), not the best case.
   - Published law-enforcement policy ("almost nothing to hand over") and an abuse-reporting path without content or identities.
   - Privacy policy, transparency page, operator list.
   - Reproducible builds and signed releases.
7. **Public release** of the Android MVP.

**MVP scope:** 1:1 chat, small groups (≤ 50), images and files, read-once by default, Android only.

**Exit criteria:** audit findings closed or publicly documented; beta meets reliability targets; release published.

---

## Phase 6 — After the MVP

Order reflects WP §11.2 step 6 and the Konzept's "Danach". Each item needs its own spec first.

1. **Linux desktop.** Key hierarchy via TPM/OS keystore, or a passphrase-derived root key with a stated weaker guarantee.
2. **iOS**, gated on two things:
   - Licensing: AGPL vs. App Store terms needs an exception from SimpleX Chat Ltd and possibly other contributors, plus legal advice. Fallbacks: EU alternative distribution, or an independent iOS core.
   - Push: visible "New message" notifications via a notification service extension, with a fixed burst of 4 fetches per circuit; Secure Enclave key hierarchy; screenshot notices delayed and batched; Universal Links + `UIPasteControl`. Document that iOS delivery is less private than Android.
3. **Multi-device (V2, §8.3).** Per-device mailboxes, QR pairing, consumed markers, 60 s sync gate, optional "single active reader", device revocation.
4. **Large groups and channels with MLS (§8.1).** OpenMLS, per-group pseudonymous credentials, no key-package directory, 2–3 group relays with a sequencer and failover, 7-day self-update commits. Open item: standardisation status of a hybrid X25519 + ML-KEM MLS cipher suite.
5. **Sender Keys** for small groups, if fan-out proves too costly.
6. **Volunteer nodes.** Only after directory governance and operator vetting are defined. No token.
7. **Optional extras** that could fund the network: large files, one-click private relays.

---

## Cross-cutting tracks (run alongside all phases)

| Track | Owner | Key steps |
|---|---|---|
| Upstream sync | Core devs | Monthly rebase on upstream tags; maintain `mujo-docs/fork-diff.md`; keep new code in separate modules |
| Operators | Infra / founders | 2 operators (Ph1) → ≥ 4 (Ph3 staging) → ≥ 6 (release); agreements; directory signing ceremony |
| Funding | Founders | Decide model by end of Ph1; needed before operator contracts and audit payment |
| Legal | External counsel | Trademark (Ph0), AGPL/App Store (before iOS), LE request policy and abuse reporting (Ph5), GDPR review of push services |
| Security process | Core devs | Threat model kept current; fuzzing of parsers; crash-injection tests for read-once; responsible disclosure policy before beta |
| Measurement | Android dev | Device lab: StrongBox, TEE-only, low-end; data/battery dashboards for Table 5 |
| Docs | All | Specs before code; white paper revised (v0.4+) whenever measured numbers replace targets |

---

## Milestone summary

| # | Month | Milestone | Gate |
|---|---|---|---|
| M0 | 2 | Prototype validated, fork decision | Go/no-go on separate product |
| M1 | 3 | Branded build on own relays, core specs done | Specs reviewed |
| M2 | 7 | Two-relay mailboxes + read-once + key hierarchy | Outage and crash-injection tests pass |
| M3 | 11 | Circuits + signed directory + cover traffic | ≥ 4 operators in staging, measurements published |
| M4 | 14 | Ticket push + new UI | Usability bar met (< 60 s, no help) |
| M5 | 18 | Audit done, Android MVP released | Audit findings closed |
| M6 | 18+ | Desktop, iOS, multi-device, MLS, volunteer nodes | Per-feature specs and licensing |

Phases 3/4 and 4/5 overlap on purpose (months 10–11, 14), as in WP §11.2.

---

## Open decisions to settle early

1. Fork vs. upstream contribution (Phase 0).
2. Name, if EUIPO/DPMA checks fail.
3. Funding model and who pays for nodes.
4. Small-group technique in the MVP (fan-out vs. Sender Keys).
5. Who signs the node directory, and how operators are vetted.
6. iOS licensing path.
7. Hybrid PQ cipher suite for MLS.

## Differences between the two source documents

| Topic | Konzept | WP v0.3 (used here) |
|---|---|---|
| MVP timeline / team | 6–9 months, 2–3 devs | 12–18 months, 3–4 devs + designer |
| MVP platforms | Android + Desktop (iOS if push ready) | Android only |
| Push | Encrypted device token per mailbox | Blind-token wake-up tickets, split service |
| Entry hop | Shared for both deposits | Separate circuit per relay |
| Read receipts | Optional "read and deleted" | Off by default, delayed and batched |
| On-device erasure | Delete key before display | Hardware EEK + table-key rotate-and-rewrap |
| Multi-device read-once | "Reading on one deletes on all" | First-reader-wins with consumed markers and sync gate |
| Cover traffic | Optional dummy packets | Fixed-rate polling schedule (Table 5) |
