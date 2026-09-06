# STILL REMAINING / NOT DONE

An honest accounting of what the four-item task ("1. add screenshots, 2. build the
genesis funding path, 3. one-command CLI copy-paste quickstart, 4. do everything
in order then summarize what is lacking") delivered, and what is genuinely not
done. Items 1-3 shipped in prior features; item 4 wired live inference, built a
bridge burn->unlock watcher, and closed small gaps. A follow-up pass (FEAT-005)
then wired that watcher into `matrixd` as a real subsystem and gave it a
persisted scan cursor. Below is what is NOT finished and why.

## What shipped (for context)

- FEAT-001: README screenshots (multi-page IA, native-first captions).
- FEAT-002: native MATRIX Treasury wired into the node; genesis applied once on
  Start; `FundAccount` RPC + `matrix fund` CLI move reward-pool MATRIX. Live loop
  transcript captured. NOTE (review v1, finding 3): `FundAccount` carries no
  per-request signature, so it now **requires the server to enforce
  authentication** (ACLs). On an ACLs-off node it returns `FailedPrecondition`
  instead of moving funds, closing the unauthenticated reward-pool drain. So
  `matrix fund` / `matrix quickstart` work against the default (ACLs-on) node with
  an `--api-key`, and reward-pool funding is simply not exposed on an open node.
- FEAT-003: `matrix quickstart` one-command zero-to-first-job flow; `matrix init`
  dev config; web click-to-copy (CopyButton) on home + /products/cli;
  listen_addr multiaddr fix so a default node boots.
- FEAT-004 (this pass):
  - Inference wired into the live node: `node.go` constructs/serves
    `inferenceapi.Server` (matrix.inference.v1) on `Inference.Addr` (default
    `0.0.0.0:9092`), gated by the same ACL auth as the market API and torn down
    in `Stop()`. Registers the GPU-free echo backend for a configurable demo
    provider. `matrix inference submit|get` CLI added. Buyer signing keys resolve
    via a `walletAccounts` resolver (in-memory + on-disk `~/.matrix` wallets).
    Proven live end to end (echo backend): see `inference-transcript.txt`.
  - Bridge burn->unlock watcher: stdlib Ethereum JSON-RPC client + always-on
    `Watcher` + `cmd/bridge-watch`, driven end to end against a LOCAL HARDHAT node
    (real `Burned` event decoded and unlocked): see `bridge-watch-transcript.txt`.
  - Web: added a click-to-copy `matrix inference` command block to the existing
    `/products/inference` page (no new route).
- FEAT-005 (follow-up pass):
  - Bridge wired into `matrixd` as a node subsystem: a config-driven `bridge:`
    section builds the `Bridge` over the node's OWN market ledger and KV store
    and runs the burn->unlock `Watcher` against it, joined cleanly in
    `Node.Stop`. Opt-in, off by default, and loud on an incomplete config. New
    accessors `GetBridge()` / `GetBridgeWatcher()`.
  - `bridge.Watcher` gained an optional persisted scan cursor
    (`WatcherConfig.Store` / `CursorKey`), namespaced per contract, so a matrixd
    restart resumes instead of re-scanning from `start_block`.
  - Proven live: real matrixd boot, real Ethereum JSON-RPC dial, real on-chain
    `Burned` event decoded, unlock credited on the node's own ledger and read
    back over the market gRPC API, restart resuming from the persisted cursor
    with no double unlock (`bridge-in-node-transcript.txt`). The
    lock -> burn -> unlock -> `Reconcile` path with a real `Bridge.Lock` is
    covered by `internal/node.TestNodeBridgeWiring_BurnUnlocksOnTheNodeLedger`.

## NOT DONE / only the user (operator) can do these

### 1. Real mainnet / testnet deployment needs the user's funded key
The Hardhat contracts (`WrappedMatrix`, `MatrixToken`) and the deploy scripts
exist and pass 38 tests locally, but deploying to Sepolia or mainnet requires a
funded deployer private key (`PRIVATE_KEY`) and an RPC URL
(`MAINNET_RPC_URL` / `SEPOLIA_RPC_URL`), all env-only. No key is present in this
sandbox and none may be committed, so no real-network deployment was performed.
Operator step: set those env vars and run `npx hardhat run scripts/deploy-*.ts
--network <net>`.

### 2. GPU-dependent local inference beyond the echo backend
The live inference demo uses the GPU-free deterministic **echo** backend, which
is what makes the whole reserve -> fulfill -> settle -> completed path testable
without hardware. Two other backends exist but were NOT exercised live here:
- **local-http** (Ollama/llama.cpp-style): needs a running local model server and
  typically a GPU. Not available in the sandbox (no GPU / no ollama).
- **openai** (provider-API proxy): reads its API key from an environment variable
  (`OPENAI_API_KEY`) only. No key is present and none may be committed, so this
  path is env-gated and unexercised. The code path is built and unit-tested with
  a stub HTTP server (`internal/inference/openai_test.go`), but a real provider
  call requires the user's key.
Operator step: run a local model server and register the local-http backend, or
set `OPENAI_API_KEY` and register the openai backend, via
`GetInference().Registry()`.

### 3. Native Tauri desktop bundle needs system libraries
The `apps/console` Tauri + React app cannot be bundled in this sandbox: the
native build needs system libs (webkit2gtk, gtk) that are not installed. The web
frontend (`apps/web`) builds and lints cleanly; only the native desktop bundle is
blocked. Operator step: install the Tauri prerequisites for the target OS and run
the Tauri bundle command.

### 4. Bridge watcher: what is automated vs. what an operator still runs
Be precise here to avoid overstating.

**Automated and tested (this pass):**
- Decoding a real on-chain `Burned` event into a `BurnEvent`
  (`internal/bridge.DecodeBurnedLog`), byte-verified against Hardhat-emitted logs.
- Ingesting those events by polling `eth_getLogs` over JSON-RPC
  (`internal/bridge.HTTPEthClient`) and driving the replay-safe unlock
  (`internal/bridge.Watcher` -> `ProcessBurn`), with confirmation-depth handling,
  range chunking, and per-id replay-safety. Unit-tested with a fake client and
  proven end to end against a **local hardhat node** (`cmd/bridge-watch`, see
  `bridge-watch-transcript.txt`).

**Now automated in `matrixd` (FEAT-005) -- these two items are DONE:**
- The watcher **runs inside `matrixd`** against the node's own ledger. A
  config-driven `bridge:` section (`contract`, `chain_id`, and a `watch` block
  with `enabled`, `rpc_url`, `start_block`, `confirmations`, `poll_interval`,
  `max_block_span`) makes `node.Start` construct the `Bridge` over the node's own
  market ledger and KV store and run the `Watcher` against it, so the unlock hits
  the same ledger that holds the locks and `Bridge.Reconcile` is a meaningful
  invariant check. `Node.Stop` joins the watcher goroutine before closing that
  ledger and store. The subsystem is opt-in and off by default; a config that
  enables `bridge.watch` without a contract, chain id, or rpc_url fails startup
  rather than coming up quietly not relaying. See
  `internal/node/bridge_watch.go`. Proven live against a local hardhat chain:
  `bridge-in-node-transcript.txt`.
- The `Watcher` **now persists its scan cursor** when given a `Store`
  (`WatcherConfig.Store` + `CursorKey`, which the node supplies as the KV store
  with a per-contract key), so a restart resumes at the next unscanned block
  instead of re-scanning from `start_block`. Correctness still does not depend on
  it: the cursor is written AFTER a span's burns are applied, so a crash re-scans
  (dedup'd by `ProcessBurn`) rather than skipping; a persisted cursor behind
  `StartBlock` is ignored; a corrupt cursor fails construction loudly; and a
  failed cursor write is reported but never stops the relay. This closes review
  v1 finding 1, which had been left as an optional optimization.

**What an operator still runs / decides (NOT automated):**
- **Multi-validator consensus on the unlock is not part of the watcher.** The
  watcher applies burns to one node's ledger; in a multi-validator deployment the
  unlock must ultimately be agreed through consensus (as compute/inference
  settlement already is). Running the watcher inside `matrixd` does **not** change
  this: the unlock is still applied by whichever node runs the watcher rather than
  through a consensus-ordered operation. Making burn->unlock consensus-ordered is
  a larger design item and is **not done**.
- The operator supplies the **RPC endpoint, the deployed WrappedMatrix address,
  the chain id, the start block, and the confirmation depth**. There is no
  automatic contract discovery. `confirmations` left unset defaults to 12 rather
  than 0, so a node pointed at a real endpoint does not release escrow on a
  reorgable block; `0` is for instant-finality dev chains only.
- **`cmd/bridge-watch` remains a diagnostic, not a deployment tool.** It applies
  burns to a throwaway ledger it seeds, so its unlock is disconnected from the
  locks backing the wrapped supply and `Reconcile` cannot be checked against it.
  Use it to inspect a block range or verify an endpoint/address before enabling
  `bridge.watch`.

### 5. Web unit tests (test runner intentionally NOT added)
`apps/web` still has **no test runner**. Adding jest/vitest + jsdom +
testing-library to a Next 15 / React 19 / Tailwind v4 project is a non-trivial
toolchain change that risks destabilizing the `next lint` / `next build` gates,
which the task explicitly warned against ("If adding a test runner risks the Next
15 / Tailwind v4 build, document it as a remaining item instead"). So per that
guidance a web test runner was **deliberately not added**. The `CopyButton`
component is a small, self-contained `'use client'` component whose behavior
(clipboard write + honest success/failure toast) is exercised manually; a proper
component test remains a remaining item pending a decision to introduce a test
runner. The Go side that the web copies commands for IS covered by real tests and
live transcripts.

### 6. Not attempted (out of scope, noted for completeness)
- Screenshot (re)capture is owned by the orchestrator, not this coder. The
  `/products/inference` page gained a click-to-copy command block (existing route,
  not a new one); the orchestrator may optionally re-screenshot it.
- No CI workflow changes were made (hard rule: `.github/workflows/*.yml` is the
  orchestrator's separate PR).

## Verification performed (all green)

- `make proto` (buf lint + generate).
- `cd services/core && go build ./... && go vet ./... && go test ./...`.
- `go test -race` on `internal/inference`, `internal/inferenceapi`,
  `internal/node`, `internal/cli`, `internal/bridge`.
- `matrix --help` shows `inference`; `matrix inference` runs.
- `cd contracts && npx hardhat test` -> 38 passing.
- `cd apps/web && corepack yarn install --frozen-lockfile && corepack yarn lint &&
  corepack yarn build` -> clean.
- Live inference job through matrixd + `matrix inference` (echo backend):
  `inference-transcript.txt`.
- Live bridge burn->unlock against a local hardhat node: `bridge-watch-transcript.txt`.
- Live in-node bridge watcher (matrixd with `bridge.watch` enabled) against a
  local hardhat node, including a restart that resumes from the persisted
  cursor: `bridge-in-node-transcript.txt`.
