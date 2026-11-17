# Phase 0 spikes

Throwaway code that answers questions the plan depends on. Each spike lives on a `spike/<name>` branch, never merges to `main`, and ends with a results file in this folder.

## S1: Two-relay replication

**Question:** How much of `smp-agent` must change so that one connection deposits each message on 2 SMP servers and the receiver drops the duplicate before the ratchet advances?

**Starting points**
- Connections already hold several queues for rotation: `primary`, `dbReplaceQueueId` in `src/Simplex/Messaging/Agent/Store.hs`.
- Receive path: `Agent.hs`, where `extSndId == prevExtSndId` yields `MsgDuplicate` (only consecutive duplicates).
- Send path: `enqueueMessageB` / `sendMessages` in `Agent.hs`.

**Steps**
1. Run 2 local `smp-server`s.
2. Create a connection whose receiver has 2 active (non-rotating) queues. Sender deposits the same encrypted envelope on both.
3. Receiver dedups by inner-packet hash before `rcDecrypt`.
4. Tests: one server down; one server delayed 5 s; reorder across servers; replay.

**Results to record:** files/functions touched, rough LOC, protocol or schema changes needed, open problems (e.g. per-relay encryption layer, acknowledgements on two queues).

## S2: Hardware key hierarchy timing

**Question:** Does rotate-and-rewrap (WP §7.1) meet the 300 ms tap-to-display target on Android?

**Devices:** one with StrongBox, one TEE-only, one low-end (≤ 3 GB RAM).

**Steps**
1. Minimal Kotlin app: create a non-exportable AES key (StrongBox, fallback TEE) as the EEK, plus a 256-bit table key and a table of 1,000 / 5,000 / 10,000 × 32-byte entries.
2. Time each step 50 times: remove entry, new table key, re-encrypt table, create new EEK, wrap, commit to SQLite in one transaction, delete old EEK.
3. Check `setRollbackResistant(true)` support, and what happens when trusted storage is full.

**Results to record:** median and p95 per device per table size, StrongBox vs. TEE, rollback-resistance support.

## S3: Cover traffic cost

**Question:** What do the WP Table 5 schedules cost in data and battery on real devices?

**Steps**
1. Android test app that sends 16 KB cells over TLS to a test server on the Table 5 schedules (chat open Wi-Fi/mobile, background Wi-Fi/mobile), with 2 circuits.
2. Run each profile for 24 h per device; read `BatteryStatsManager` / Android battery usage and traffic stats.
3. Check whether background 60 s polling survives Doze and app standby buckets without a foreground service.

**Results to record:** MB per hour/day, battery % per day, Doze behaviour. Compare with the WP estimates (24 MB/h, 47 MB/day, 95 MB/month).
