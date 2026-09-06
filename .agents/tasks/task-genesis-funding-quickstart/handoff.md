# Handoff - task-genesis-funding-quickstart (native genesis funding, quickstart, inference + bridge)

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

1. **Bridge watcher cursor** - dead `watcherCursorKey` constant removed; comments now state the
   watcher always resumes from `StartBlock` and re-scans harmlessly (`ProcessBurn` dedups by burn id).
2. **bridge-watch integration claim** - header softened; matrixd does NOT run an in-node Watcher,
   noted as a remaining item (`STILL-REMAINING.md`).
3. **FundAccount unauthenticated drain (most important)** - `marketapi.Service.authEnforced` is set
   from `cfg.Auth != nil` in `NewServer`; `FundAccount` returns `FailedPrecondition` on an ACLs-off
   node, so it cannot be used to move reward-pool MATRIX for an unauthenticated caller. Regression
   tests `TestFundAccount_RejectedWithoutAuth` and `TestFundAccount_RejectedWithoutCredentials`
   would fail if the gate were removed; legitimate `fund`/`quickstart` paths still pass with `--api-key`.
4. **Inference web one-liner** - now a fund → register → submit block, flags matched to the shipped
   CLI, with an ACL/`--api-key` note.
5. **Wallet key pairing** - derive-and-compare (`priv.Public()` vs stored pubkey) added to
   `cli/wallet.go loadWallet` (hard error) and `node/inference_accounts.go loadWalletAccount`
   (resolver skip); test `TestWalletAccounts_RejectsMismatchedKeyPair`.

## Verification (re-run green by orchestrator after fix)

`make proto`; `services/core` go build/vet/test `./...`; `go test -race` on
`internal/{bridge,marketapi,node,cli}`; `apps/web` lint+build clean.
Known pre-existing race-only flake (unrelated): `internal/consensus`
`TestMultiNodeNoDivergenceUnderTightTimeout`.

## Branch / merge state

The seven commits above are on `origin/main`. (An earlier revision of this file said local `main` was
7 commits ahead and nothing had been pushed; that is no longer true.) Follow-up work continues on
`claude/handoff-md-checklist-s0ptw1`.

## Follow-up pass: FEAT-005, in-node bridge watcher (DONE)

The two follow-ups this handoff listed as optional are now implemented, because the first one turns a
half-wired feature into a usable one and the second is what makes the first deployable:

- **Config-driven in-matrixd `bridge.Watcher` wiring** - a `bridge:` config section
  (`contract`, `chain_id`, `watch.{enabled,rpc_url,start_block,confirmations,poll_interval,max_block_span}`)
  makes `node.Start` construct the `Bridge` over the node's own market ledger and KV store and run the
  burn->unlock `Watcher` against it; `node.Stop` joins the goroutine before closing that ledger and
  store. Opt-in and off by default; an incomplete `bridge.watch` config fails startup instead of
  coming up quietly not relaying. New file `internal/node/bridge_watch.go`, accessors
  `Node.GetBridge()` / `Node.GetBridgeWatcher()`.
- **Persisted watcher cursor** - `WatcherConfig.Store` / `CursorKey` (the node passes its KV store
  with a per-contract key), so a matrixd restart resumes at the next unscanned block. Without it an
  in-node watcher would re-scan the whole range from `start_block` on every restart, which for a
  contract deployed far behind head is unbounded work. Correctness still never depends on it: the
  cursor is written after a span's burns are applied, a cursor behind `StartBlock` is ignored, a
  corrupt cursor fails construction loudly, and a failed write is reported but does not stop the relay.

Verification: `go build`/`go vet`/`go test ./...` clean; `go test -race` on `internal/{bridge,node}`
clean; 12 new tests (4 cursor-persistence tests in `internal/bridge`, 8 in the new
`internal/node/bridge_wiring_test.go`), mutation-checked by disabling the feature and confirming the
persistence tests fail. Proven live against a local hardhat chain -
real matrixd boot, real JSON-RPC dial, real on-chain `Burned` event decoded, unlock credited on the
node's own ledger and read back over the market gRPC API, and a restart resuming from the persisted
cursor with no double unlock: `bridge-in-node-transcript.txt`.

Stale docs corrected in the same pass: `cmd/bridge-watch` and `internal/bridge/doc.go` both claimed no
in-node watcher existed; `contracts/README.md` gained an operator section for `bridge.watch` and its
burn description no longer says log ingestion is operator-driven.

## Remaining follow-ups (still not done)

- **Consensus-ordered burn->unlock.** The watcher is a per-node relayer; running it inside matrixd
  does not make the unlock consensus-ordered. Tracked in `STILL-REMAINING.md` as a larger design item.
- **`apps/web` test runner** - still deliberately absent (see `STILL-REMAINING.md` item 5).
- Operator-only items (real network deployment, GPU/provider-API inference backends, Tauri bundle)
  remain env/hardware-gated.
