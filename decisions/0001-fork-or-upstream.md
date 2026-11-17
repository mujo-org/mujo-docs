# ADR 0001: Fork SimpleX or contribute upstream

- Status: **open**
- Deadline: end of Phase 0 (week 8)
- Sources: White Paper v0.3 §11, §12 ("Differentiation from SimpleX", "App Store licensing")

## Context

Mujo's additions over SimpleX are two-relay replication, separate-circuit 2-hop routing with cover traffic, a signed node directory, wake-up tickets, hardware-backed read-once storage and (later) multi-device and MLS groups. Some of these may fit SimpleX's own plans:

- `mujo-mq/rfcs/2025-07-15-multi-device.md` proposes multi-device via ratchet state stored on the router. That approach conflicts with Mujo's per-device mailbox design (WP §8.3).
- SMP already has private routing through a sender-chosen proxy (protocol v8). It's one forwarding hop, not separate circuits per relay.
- The agent already holds several queues per connection for queue rotation (`primary`, `dbReplaceQueueId` in `Agent/Store.hs`).

## Options

| Option | For | Against |
|---|---|---|
| A. Separate product (fork) | Full control over read-once default, UX, network policy | Upstream sync cost, Haskell staffing, AGPL/App Store problem for iOS |
| B. Contribute upstream | No sync cost, existing user base, existing operators | SimpleX may reject read-once-by-default or the network model; slower |
| C. Hybrid | Upstream protocol parts (replication, circuits), keep product layer in fork | Two review processes; still a fork |

## Questions for SimpleX Chat Ltd

1. Would you accept multi-relay replication of receive queues upstream?
2. Would you accept separate per-relay circuits and fixed-rate fetching upstream?
3. Are you planning per-device mailboxes, or only the router-stored ratchet approach of the multi-device RFC?
4. Would you grant an App Store licence exception for a fork's iOS build? What are the contributor terms (CLA) for code already in the repos?
5. Could Mujo nodes join the SimpleX Network Consortium, or interoperate with its relays?

## Decision

_Pending._

## Consequences

_To be filled once decided. If B or C, ROADMAP.md becomes a contribution plan for the affected phases._
