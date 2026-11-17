# S1 results: two-relay replication

Status: **code analysis done**, prototype not started. Base: `mujo-mq@9ffd36ed`.

Question: how much of `smp-agent` must change so that one connection deposits each message on 2 SMP servers and the receiver drops the duplicate before the ratchet advances (WP §6.1, §7.2 rule 4)?

**Short answer:** much less than expected. Wire formats and most of the send/receive machinery already handle several queues per connection, but the handshake only ever creates one queue. The work is in the handshake, in queue rotation and in one receive-path check. No protocol encoding changes are needed for the basic case.

## What already exists

| Need (WP) | Existing code | Location (`src/Simplex/Messaging/`) |
|---|---|---|
| Invitation carries 2 relay addresses (§4.1) | `ConnReqUriData.crSmpQueues :: NonEmpty SMPQueueUri`, URI `smp=q1;q2` | `Agent/Protocol.hs:1898`, `:1242` |
| Joiner's reply carries 2 queues | `AgentConnInfoReply (NonEmpty SMPQueueInfo) ConnInfo` | `Agent/Protocol.hs:944` |
| Connection holds several queues | `DuplexConnection ConnData (NonEmpty rq) (NonEmpty sq)` | `Agent/Store.hs:406` |
| Sender deposits on every relay | `submitSentMsg` sends to the first send queue and passes all other *active* ones to `enqueueSavedMessageB`; one delivery worker per queue | `Agent.hs:2527`, `:2591`, `:2614` |
| Same message must look different per relay (Table 4) | Each delivery is wrapped in a per-queue `agentCbEncrypt sq` box (queue e2e DH), and the server adds its own per-queue layer to the recipient | `Agent/Client.hs:2136` |
| Receiver identifies the same message on both relays | Inner ratchet ciphertext is deterministic per message: `rcEncryptMsg` uses the IV from the message key and the stored encrypted header, so `encryptedMsgHash = sha256 encAgentMessage` is identical on both copies | `Crypto/Ratchet.hs:962`, `Agent.hs:3749` |
| Duplicate is dropped | Second copy fails ratchet decryption (`CERatchetDuplicateMessage` → `A_DUPLICATE`, or `A_CRYPTO`), then `checkRcvMsgHashExists` finds the hash and the message is acked silently | `Agent.hs:3785–3830` |
| Hash kept longer than relay retention (7 days) | `storedMsgDataTTL = 21 days` | `Agent/Env/SQLite.hs:252` |
| Relays from different operators | `getNextServer_` takes a set of operators/hosts to exclude | `Agent/Client.hs:2587` |
| Extra receive queues become Active | First message on a non-primary queue sets it Active | `Agent.hs:3736–3738` |

## What is missing

1. **Handshake creates one queue on each side.** Every consumer reads only the head of the list:
   - `newRcvConnSrv` (`Agent.hs:1396`) creates one receive queue for the invitation.
   - Join paths take `crSmpQueues = q :| _` (`Agent.hs:1110`, `:1599`, `:2282`).
   - The joiner replies with `replyQInfo :| []` (`Agent.hs:1623`, `:4463`).
   - `connectReplyQueues` takes `(qInfo :| _)` (`Agent.hs:4403`), and the invitation path takes `(replyQueue :| _)` (`Agent.hs:4268`).
2. **Dedup happens after a ratchet decrypt attempt, not before.** WP §7.2 wants the hash check first. Today a duplicate costs a failed decryption, increments `recvDuplicates`/`recvCryptoErrs` stats and depends on the `A_CRYPTO` fallback not triggering ratchet re-sync. Fix: in the `AgentMsgEnvelope` branch, check `checkRcvMsgHashExists db connId encryptedMsgHash` before `agentClientMsg` and ack directly. This needs the check and the insert to happen under the same connection lock (`lockConnForUpdate`), otherwise two copies arriving at the same moment on two subscriptions can race.
3. **Queue rotation assumes one active queue.** `switchConnection` / `QADD`/`QKEY`/`QUSE`/`QTEST` replace *the* queue via `dbReplaceQueueId`. With 2 replicas, rotation must replace one replica at a time; that is also the WP's relay-replacement mechanism ("if a relay fails, the recipient replaces it").
4. **HELLO and confirmation on both queues.** `secureConfirmQueue` runs per send queue. The joiner must secure and confirm both queues, and the initiator must accept the confirmation from either one.
5. **Ack semantics.** Each relay copy needs its own `ACK`, including the dropped duplicate, or the relay keeps redelivering until its 7-day expiry.
6. **Version gating.** A new agent version must advertise "replicated queues", so that a peer on the old version still gets a single queue.

## Proposed prototype (branch `spike/replication` in `mujo-mq`)

1. Add a `replicas :: Int` (default 1) to agent config. When it's 2, `newRcvConnSrv` creates a second receive queue on a server picked with `getNextServer_` excluding the first queue's operator, and the invitation carries both.
2. Join: create one send queue per invitation queue, secure/confirm each, create 2 reply queues and send both in `AgentConnInfoReply`.
3. `connectReplyQueues`: connect all reply queues, not just the head.
4. Receive: hash pre-check under the connection lock before `agentClientMsg`, ack the duplicate.
5. Tests in `tests/AgentTests/FunctionalAPITests.hs` style, with 2 test servers on different ports:
   - messages delivered with both servers up, no duplicate `MSG` events;
   - server B down: delivery via A only; server B back up: no duplicate after redelivery;
   - server A delayed 5 s: order and integrity (`MsgOk`) preserved;
   - replay of an old envelope on either queue: dropped.

Out of scope for the spike: replica replacement through rotation (item 3), version negotiation (item 6), notifications for 2 queues.

## Risks found

- **Integrity bookkeeping.** `checkMsgIntegrity` uses `prevExtSndId`; with the hash pre-check, the second copy never reaches it, which is what we want. Without the pre-check, a late duplicate arriving after newer messages would be classified `MsgBadId` and surfaced as an error.
- **Read-once interaction (Phase 2).** Once per-message keys are erased after reading, the receiver can't re-derive anything from a late copy. The hash pre-check is then the only correct dedup path, so it must land before the read-once work.
- **Hash table growth.** One row per received message for 21 days. Fine on phones, but measure it in the prototype.
