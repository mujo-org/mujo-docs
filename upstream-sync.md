# Upstream sync policy (draft)

Applies to `mujo-mq` (upstream `simplex-chat/simplexmq`) and `mujo-chat` (upstream `simplex-chat/simplex-chat`).

## Remotes and branches

- `origin`: `mujo-org/<repo>`. `upstream`: `simplex-chat/<repo>`.
- `upstream-master` mirrors upstream `master` unmodified.
- `main` carries Mujo changes on top of the latest synced upstream tag.

## Cadence

- Sync on every upstream release tag, at least monthly.
- Security fixes are synced within 48 hours of upstream publishing them.

## Method

1. Merge (not rebase) the upstream tag into `main`, so the published history stays stable.
2. Resolve conflicts and run the reduced CI build with all tests: put a `ci-tests` block containing `/` in the PR body (`scripts/ci/run-tests.sh` passes each line as `-m <pattern>`; without the block the reduced build runs no tests).
3. Bump `mujo-chat`'s `source-repository-package` pin for `mujo-mq` in the same sync.
4. Update [fork-diff.md](fork-diff.md) if a Mujo change had to be reworked.

## Keeping the diff small

- New Mujo code goes into new modules (e.g. `Simplex.Messaging.Agent.Replication`). Upstream modules get only minimal hooks.
- Protocol changes are versioned through SMP/agent version negotiation, never by changing existing encodings.
- Follow upstream style (`fourmolu`, `contributing/CODE.md`) so changes could be proposed upstream.
