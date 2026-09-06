# Handoff — task-genesis-funding-quickstart (native genesis funding, quickstart, inference + bridge)

## Status: APPROVED (review pass v2)

The v2 re-review of fix commit `86c7d39` confirms all five v1 findings are resolved with
no regressions. Verdict: **APPROVED**. Full review:
`.agents/tasks/task-genesis-funding-quickstart/2026-09-06-125913-review.md`.

## What shipped (7 commits on `main`, ahead of `origin/main`)

- `9895e0f` docs: multi-page IA README screenshots, native-first captions
- `ba82f7e` feat: native MATRIX treasury genesis + reward-pool funding path
- `b2670e0` feat: one-command `matrix quickstart` + click-to-copy web start commands
- `dc36ee5` feat: wire inference service into the node + `matrix inference` CLI
- `05a22b5` feat(bridge): always-on burn→unlock watcher (tested against local hardhat)
- `1533f1b` feat(web): click-to-copy inference command on the inference page
- `86c7d39` fix: address review v1 findings (fund auth, wallet pairing, watcher/inference docs)

## v1 findings and how they were resolved (all verified in v2)

1. **Bridge watcher cursor** — dead `watcherCursorKey` constant removed; comments now state the
   watcher always resumes from `StartBlock` and re-scans harmlessly (`ProcessBurn` dedups by burn id).
2. **bridge-watch integration claim** — header softened; matrixd does NOT run an in-node Watcher,
   noted as a remaining item (`STILL-REMAINING.md`).
3. **FundAccount unauthenticated drain (most important)** — `marketapi.Service.authEnforced` is set
   from `cfg.Auth != nil` in `NewServer`; `FundAccount` returns `FailedPrecondition` on an ACLs-off
   node, so it cannot be used to move reward-pool MATRIX for an unauthenticated caller. Regression
   tests `TestFundAccount_RejectedWithoutAuth` and `TestFundAccount_RejectedWithoutCredentials`
   would fail if the gate were removed; legitimate `fund`/`quickstart` paths still pass with `--api-key`.
4. **Inference web one-liner** — now a fund → register → submit block, flags matched to the shipped
   CLI, with an ACL/`--api-key` note.
5. **Wallet key pairing** — derive-and-compare (`priv.Public()` vs stored pubkey) added to
   `cli/wallet.go loadWallet` (hard error) and `node/inference_accounts.go loadWalletAccount`
   (resolver skip); test `TestWalletAccounts_RejectsMismatchedKeyPair`.

## Verification (re-run green by orchestrator after fix)

`make proto`; `services/core` go build/vet/test `./...`; `go test -race` on
`internal/{bridge,marketapi,node,cli}`; `apps/web` lint+build clean.
Known pre-existing race-only flake (unrelated): `internal/consensus`
`TestMultiNodeNoDivergenceUnderTightTimeout`.

## Branch / merge state

All work is on local `main` (7 commits ahead of `origin/main`); there is no separate feature branch —
the "merge to main" is already reflected because the commits live directly on `main`. No local feature
branches exist to clean up (`git branch` shows only `main`). The remote branch
`origin/ci/consolidate-workflows` is unrelated to this task and was left untouched. Nothing was pushed
(no PR; local main is intentionally ahead of origin per the review setup). Pushing `main` to `origin`
is left to the operator, since force/publish actions were not requested and this repo has no PR flow here.

## Follow-ups (non-blocking, out of scope for this task)

- Optional config-driven in-matrixd `bridge.Watcher` wiring (tracked in `STILL-REMAINING.md`).
- Optional persisted watcher cursor to skip redundant re-scans across restarts (correctness never
  depends on it; current re-scan is harmless via burn-id dedup).
