# dcrd Architecture Model (Pre-Launch Review Reference)

Scope: whole-repository architecture model of this checkout, for internal
pre-launch review. All `file:line` references are relative to the repository
root and were read at the commit below. Statements marked **(inferred)** are
derived from reading code but not exercised; anything not confirmed is marked
`VERIFICATION PENDING` or listed under Open Questions. This document makes no
correctness verdicts.

---

## Commit + Build/Test

### Commit

```
$ git log --oneline -1
17dd9f4 netsync: Reset sync height on new sync peer.
```

- Version string: `2.2.0-pre` (`internal/version/version.go:60`).
- Clone is **shallow** (105 commits; boundary `f2cdd93`, a squashed import of
  the whole tree). No git tags. `git log -S`/blame cannot date code older than
  the import.
- Linear history; last release notes are `docs/release-notes/release-notes-2.1.6.md`
  (885ea18). 68 commits follow it up to HEAD.

### Module layout

The repo is a multi-module workspace. Root module `github.com/decred/dcrd`
(`go.mod`, `go 1.25.0`) uses a `replace` block (`go.mod:64-93`) mapping every
in-repo module to its local directory, so the root build always compiles HEAD
sources regardless of the `require` versions.

| Module (`github.com/decred/dcrd/...`) | Dir | Role |
|---|---|---|
| (root) | `.` | `dcrd` binary: `server.go`, `config.go`, `rpcadaptors.go`, plus `internal/*` (chain, mempool, mining, netsync, rpcserver, connmgr, fees, ...) |
| `blockchain/v5` | `blockchain` | **Only** test generators: `chaingen`, `fullblocktests`. Consensus code is in root `internal/blockchain`. |
| `blockchain/stake/v5` | `blockchain/stake` | Stake tx rules, ticket DB, lottery, ticket treap |
| `blockchain/standalone/v2` | `blockchain/standalone` | Stateless consensus helpers (tx sanity, subsidy, PoW, merkle, treasury windows) |
| `txscript/v4` | `txscript` | Script engine, sigcache, `stdaddr`, `stdscript`, `sign` |
| `wire` | `wire` | P2P message encoding/decoding |
| `peer/v4` | `peer` | Per-peer protocol state machine |
| `database/v3` | `database` | DB interface + `ffldb` driver |
| `mixing` | `mixing` | StakeShuffle messages, `mixpool`, `mixclient` |
| `addrmgr/v4`, `chaincfg/v3`, `chaincfg/chainhash`, `dcrutil/v4`, `gcs/v4`, `dcrec/*`, `crypto/*`, `math/uint256`, `container/*`, `hdkeychain/v3`, `bech32`, `certgen`, `dcrjson/v4`, `rpc/jsonrpc/types/v5`, `rpcclient/v9` | same-named dirs | Supporting libraries |

Version skew to be aware of (resolved by `replace` for the root build only):
root requires `blockchain/stake/v5 v5.0.2` (`go.mod:11`) although 7e3dd85
prepared v5.0.3; most submodules still require `wire v1.7.x` while root uses
v1.8.0. Submodule tests (`cd <module> && go test`) resolve **published**
versions except where a module has its own `replace` (only `peer` → `../wire`
and `rpcclient`). The root `replace github.com/decred/dcrd/limits => ./limits`
(`go.mod:85`) points at a non-existent directory; nothing requires that path
(`dcrd.go:20` imports `internal/limits`), so it has no build effect.

### Core vs supporting packages

| Class | Packages |
|---|---|
| **Core processing** | `internal/blockchain` (validation, chain selection, UTXO, stake integration, agendas, treasury), `blockchain/stake`, `blockchain/standalone`, `txscript`, `wire`, `peer`, `internal/netsync`, `internal/mempool`, `internal/mining`, `database/ffldb`, `mixing/mixpool`, `server.go` (glue), `internal/rpcserver` |
| **Supporting** | `chaincfg` (params), `internal/connmgr`, `addrmgr`, `internal/fees`, `internal/blockchain/indexers`, `internal/ratelimit`, `internal/limits`, `internal/progresslog`, `internal/staging/primitives`, `internal/version`, `dcrutil`, `gcs`, `dcrec/*`, `crypto/*`, `container/*`, `math/uint256`, `rpcclient`, `dcrjson`, `cmd/*` |

External dependencies (root `go.mod:5-61`): `goleveldb` (metadata + UTXO DB),
`gorilla/websocket` (RPC ws), `jessevdk/go-flags` (config), `decred/slog`,
`jrick/logrotate`, `jrick/bitset`, `dchest/siphash`, `lukechampine.com/blake3`,
`decred/go-socks`, `golang.org/x/{net,sys,term,crypto}`, `cspp/v2` and
`sntrup4591761` (mixing, indirect), `dcrtest/dcrdtest` (integration tests).

### Build and test

```sh
go build ./...                      # root binary + internal packages
go install . ./cmd/...              # dcrd, addblock, gencerts, promptsecret
./run_tests.sh                      # every module: go test -short -tags rpcserver ./...  then lint.sh (outside CI)
go test ./internal/blockchain/...   # consensus, incl. TestFullBlocks (internal/blockchain/fullblocks_test.go:167)
go test ./internal/mempool/ ./internal/mining/... ./internal/netsync/ ./internal/rpcserver/
(cd blockchain/stake && go test ./...)
(cd txscript && go test ./...); (cd wire && go test ./...); (cd peer && go test ./...)
(cd database && go test ./...); (cd mixing && go test ./...)
go test -tags rpctest ./internal/integration/rpctests/   # full-binary RPC tests (needs built dcrd); NOT run by run_tests.sh/CI
```

- CI (`.github/workflows/go.yml`): Go 1.26 and 1.27 matrix, `go build ./...`,
  `run_tests.sh`; golangci-lint v2.13.1 per module on Go 1.26. README requires
  Go 1.26/1.27 while root `go.mod` says `go 1.25.0`; `debug.go` is
  `//go:build go1.26` (GODEBUG defaults) and is silently skipped on 1.25.
- `-tags rpcserver` currently gates nothing (no `//go:build rpcserver` file).
- Test conventions:
  - **Full-block tests**: `blockchain/fullblocktests/generate.go` produces
    `TestInstance`s that `internal/blockchain/fullblocks_test.go` runs.
  - **chaingen harness**: `internal/blockchain/common_test.go:641`
    (`newChaingenHarness`, with AcceptHeader/AcceptBlock/RejectBlock/ExpectTip)
    is used by process, agenda, validate, treasury, utxo and indexer tests.
  - **Fake chain/node helpers**: in the same file, for threshold-state tests.
  - **Mempool `fakeChain`**: `internal/mempool/mempool_test.go`.
  - **testdata**: bz2 chain snapshots in `internal/blockchain/testdata`.
- **No native Go fuzz tests** (`func Fuzz` absent repo-wide). There are 39 files
  with benchmarks.

---

## Subsystems

### Network and message processing (`peer/`, `wire/`, `server.go`, `internal/connmgr`, `addrmgr`)
- **Responsibility:** peer handshake, framing, decode, dispatch, ban/relay.
  - `Peer.Run` starts stall, in, queue and out handlers (`peer/peer.go:2361-2385`).
  - Handshake: `version` must be first, with a 30s timeout and
    `min(local, remote)` protocol version (`peer/peer.go:1996-2049`, `:2348`).
  - A duplicate `version`/`verack` disconnects (`:1317-1327`).
- **Decode:** `wire.ReadMessageN` (`wire/message.go:362-498`) checks, in order:
  32 MiB cap (`:411`), magic (`:419`), ASCII command (`:425`), per-type
  `MaxPayloadLength` before reading (`:437-446`), checksum (`:471`), `BtcDecode`
  (`:482`), trailing bytes (`:488-492`).
- **Dispatch:** `processInboundMessage` runs a type switch that calls
  `MessageListeners` synchronously on the peer's `inHandler` goroutine.
  - The server wires the listeners in `newPeerConfig` (`server.go:2208`).
  - Any `wire.ErrorCode` read error triggers a **ban** (`server.go:1851-1859`).
  - Ban scoring is `addBanScore` (`server.go:907-937`, threshold 100).
- **Data owned:**
  - per-peer known-inventory APBF (`peer/peer.go:51,55`);
  - output queue capped at 40 MiB (`:1848-1857`);
  - server `peerState` (banned map `server.go:2765`), `recentlyAdvertisedTxns`, and the getdata quotas (`server.go:120-121, 1440-1457`).
- **Boundaries:**
  - connmgr accepts the connection, then the per-group rate limit and the `MaxSameIP`/`MaxPeers` semaphores apply (`internal/connmgr/connmanager.go:1611-1709`).
  - The ban check runs after the accept (`server.go:2187`).
  - Block, tx and mix messages hand off to netsync (`server.go:1348, 1356-1371, 1754`).

### Transaction processing and mempool (`internal/mempool`, `internal/fees`)
- **Responsibility:** policy-enforced admission of unmined txs.
  - Single entry point `ProcessTransaction` (`internal/mempool/mempool.go:2360`) → `maybeAcceptTransaction` (`:1400`).
  - It takes `mp.mtx` for writes for the whole call (`:2369`).
- **Data owned (`internal/mempool/mempool.go:410-436`):** `pool`, `outpoints`, `staged`/`stagedOutpoints` (tickets that spend mempool outputs), `orphans`/`orphansByPrev`, `tspends`, `voteTrack` (own RWMutex, `:269`), `miningView`.
- **Ordering:** none is kept in the pool; ordering is computed at template time.
- **Eviction is by rule, not by size:**
  - expiry (`PruneExpiredTx`) and stake pruning (`pruneStakeTx`, `:2214`);
  - orphan TTL and random eviction (`:550-593`, default 100 orphans, 15 min TTL);
  - **no global pool size cap** was found.
- **Fees:** min relay fee `calcMinRequiredTxRelayFee` (`internal/mempool/policy.go:76`); fee estimator `fees.Estimator.AddMemPoolTransaction` (`internal/fees/estimator.go:764`).
- **No RBF:** any outpoint conflict is rejected (`checkPoolDoubleSpend`, `internal/mempool/mempool.go:1032-1062`).
- **Dependencies:** chain callbacks wired in `server.go:4040-4140`:
  - `FetchUtxoView`, `CalcSequenceLock`, `HeaderByHash`;
  - `WinningTicketsByHash` → `chain.LotteryDataForBlock` (`server.go:4082`);
  - agenda flags;
  - mixpool hooks (`server.go:4132`).

### Block construction (`internal/mining`)
- **Responsibility:** template generation `NewBlockTemplate` (`internal/mining/mining.go:1170`), based on:
  - a snapshot taken from `TxSource.MiningView()` (`internal/mining/interface.go:30-59`, `internal/mining/mining.go:1307`);
  - a priority heap `txPQByStakeAndFee` (`internal/mining/txpriorityqueue.go:89-149`): votes, then auto-revocations, then tickets, then others, then by fee/kB, then by priority.
- **Selection:**
  - ancestor bundles (`mining_view.go`, tracking limit 25);
  - per-bundle `CheckTransactionInputs` and `ValidateTransactionScripts` re-run against the template's UTXO view (`internal/mining/mining.go:1752, 1762`);
  - size and sigop limits (`:1690-1713`);
  - stake-tree assembly and votebits majority (`:1911-2059`);
  - fee scaling to the coinbase (`:2102-2157`).
- **Validation:** `CheckConnectBlockTemplate` (`internal/mining/mining.go:2338` → `internal/blockchain/validate.go:4648`) runs all checks except PoW.
- **Background generator:** `bgblktmplgenerator.go` regenerates on chain and vote events (`:45-81`) and timers (`:24-40`). It may call `ForceHeadReorganization`. A cancelled generation only discards its result (`:704-771`). It is enabled only with mining addresses (`server.go:4171`).
- **CPU miner:** `internal/mining/cpuminer` submits through `SyncManager.ProcessBlock` (`internal/netsync/manager.go:2197`).

### Blockchain state management (`internal/blockchain`)
- **Locks:** `processLock` serialises processing. `chainLock` guards state and is released only around notifications (`internal/blockchain/chain.go:169-176, 763-769`).
- **Layered validation**, each layer with a different state dependency:
  1. header sanity (`internal/blockchain/validate.go:768`)
  2. header positional (`:1239`)
  3. data preconditions (`:1978`)
  4. data sanity (`:881`)
  5. data positional (`internal/blockchain/process.go:287-338`)
  6. context (`internal/blockchain/validate.go:2104`)
  7. connect (`checkConnectBlock`, `:4370`)
- **Chain selection** is purely candidate-driven: `blockIndex.FindBestChainCandidate` (`internal/blockchain/blockindex.go:1385`) ordered by `betterCandidate` (`:446`): more work, then data present, then received order, then hash.
- **Reorgs:**
  - `reorganizeChain` (`internal/blockchain/chain.go:1269`) wraps `reorganizeChainInternal` (`:1039`): detach using the spend journal, then attach with validation.
  - On failure it retries with the next best candidate (`:1326-1338`).
- **UTXO transitions:**
  - happen in `UtxoViewpoint.connectBlock`/`disconnectBlock` (`internal/blockchain/utxoviewpoint.go:669, 725`);
  - parent disapproval is handled by `disconnectDisapprovedBlock` (`:623`);
  - changes are committed only in `connectBlock`/`disconnectBlock` (`internal/blockchain/chain.go:586, 807`).
- **Outputs:** synchronous notifications `NTNewTipBlockChecked`, `NTBlockAccepted`, `NTBlockConnected`/`Disconnected`, `NTReorganization` and `NTNewTickets` go to `server.handleBlockchainNotification` (`server.go:2872`).

### Stake processing (`blockchain/stake`, `internal/blockchain/stakenode.go`, `stakeversion.go`, `difficulty.go`)
- **Ticket lifecycle:**
  - Purchase: SStx shape `CheckSStx` (`blockchain/stake/staketx.go:713`); price `checkProofOfStake` (`internal/blockchain/validate.go:655`); inputs `checkTicketPurchaseInputs` (`:2665`).
  - The ticket becomes live at +TicketMaturity (`internal/blockchain/stakenode.go:21-38`, `blockchain/stake/tickets.go:676`).
  - Outcome: voted, missed (`blockchain/stake/tickets.go:539-597`) or expired (`:601-640`).
  - Revoked: SSRtx (`CheckSSRtx`, `blockchain/stake/staketx.go:1192`; treap move `blockchain/stake/tickets.go:643`).
- **Lottery:**
  - IV is the hash of the serialised header (`internal/blockchain/blockindex.go:307`, `blockchain/stake/lottery.go:38`);
  - indices from `findTicketIdxs`/`fetchWinners` (`blockchain/stake/lottery.go:156, 188`) over the treap's in-order index (`blockchain/stake/internal/tickettreap/common.go:93`);
  - the `finalState` commitment is checked at `internal/blockchain/validate.go:1553`.
- **Stake node cache:**
  - `fetchStakeNode` (`internal/blockchain/stakenode.go:97`) connects from the parent, or walks tip→fork using DB undo data keyed **by height**, then replays the side chain;
  - pruned to 288 in-memory nodes (`internal/blockchain/chain.go:39, 522`).
- **Vote rules:**
  - voters in [TicketsPerBlock/2+1, TicketsPerBlock] (`internal/blockchain/validate.go:848-865`);
  - every vote commits to the parent (`:2259-2276`);
  - Voters equals the vote count (`:2326`);
  - the approval bit matches the majority (`:2335-2344`);
  - eligibility via `checkTicketRedeemers` (`:1745`).
- **Stake difficulty:** `calcNextRequiredStakeDifficulty` (`internal/blockchain/difficulty.go:860`) chooses V1 or V2 by agenda. It is enforced against `header.SBits` (`internal/blockchain/validate.go:1513`).
- **Stake version:** `calcStakeVersion` (`internal/blockchain/stakeversion.go:251`) is enforced in the header (`internal/blockchain/validate.go:1525-1535`).

### Treasury (`internal/blockchain/treasury.go`, `blockchain/stake/treasury.go`, `blockchain/standalone/treasury.go`)
- **Tx types:** treasurybase, TAdd and TSpend are recognised only for `tx.Version >= TxVersionTreasury` (`blockchain/stake/staketx.go:1272-1297`). Shape checks are at `blockchain/stake/treasury.go:54/140/242`.
- **Context checks:** `checkTransactionContext` (`internal/blockchain/validate.go:412`); the treasurybase must be the first stake tx and encode the height (`:2364-2378`); at most 20 TAdds per block (`:68`).
- **TSpend inclusion:**
  - only on a TVI block inside the window (`internal/blockchain/validate.go:2504-2527`);
  - `tspendChecks` (`:4263`) checks the window, `checkTSpendExists` (`internal/blockchain/treasury.go:943`), `checkTSpendHasVotes` (`:1114`) and `checkTSpendsExpenditure` (`:899`);
  - the policy cap is `maxTreasuryExpenditure` (`:849`), which chooses DCP0006, DCP0007 or DCP0013.
- **Balance:**
  - a per-block-hash `treasuryState` (`internal/blockchain/treasury.go:113, 366, 406`) written in `connectBlock` (`internal/blockchain/chain.go:681-692`);
  - not deleted on disconnect (hash-keyed, ancestor-checked).

### Synchronization (`internal/netsync/manager.go`)
- **No event loop goroutine:** handlers (`OnBlock` `:1216`, `OnTx` `:968`, `OnHeaders` `:1506`, `OnInv` `:1880`, `OnMixMsg` `:1046`) run synchronously on each peer's `inHandler` goroutine, and shared state is guarded by mutexes.
- **Headers-first:**
  1. A single sync peer, chosen by the highest `LastBlock` (`:590-645`), serves `startInitialHeaderSync` (`:655-688`).
  2. Headers go through `chain.ProcessBlockHeader` (`:1620`).
  3. Blocks are then downloaded from `chain.PutNextNeededBlocks` with at most 16 in flight per peer (`fetchNextBlocks` `:490-547`).
- **Gating:** `requestedBlocks`/`requestedTxns` maps (≤ `MaxInvPerMsg`, `:78-82`); unrequested blocks disconnect the peer (`:1222-1231`).
- **Sync height:** drives `IsCurrent` (`:1107-1127`), the getblockchaininfo RPC, and fetch offsets. It is reset at `:682` (17dd9f4), `:1283` and `:1629`.
- **Post-block hooks** (main-chain blocks): `PruneStakeTx`, `PruneExpiredTx`, `rejectedTxns.Reset`, mixpool pruning (`:1348-1357`).

### Persistence (`database/`, `internal/blockchain/chainio.go`, `utxocache.go`, `utxobackend.go`, `indexers/`)
- **`database.DB` interface:**
  - Defined at `database/interface.go:439-480`; `Tx` at `:226-420`.
  - It has a single registered driver, `ffldb` (`database/ffldb/driver.go:81`): flat block files plus leveldb metadata behind a write cache that flushes at 100 MB or 5 min (`database/ffldb/dbcache.go:22,27,530-545`).
  - Commits go through `writePendingAndCommit` (`database/ffldb/db.go:1684-1735`), which rolls back flat files if the commit fails.
- **Chain metadata buckets** (`internal/blockchain/chainio.go:52-110`): block index `blockidxv3` (`dbPutBlockNode`), `chainstate` (best state), `spendjournalv3`, `gcsfilters`, `hdrcmts`, and the treasury buckets. The block index is flushed lazily (`internal/blockchain/blockindex.go:1408-1435`, `internal/blockchain/chain.go:1469`).
- **UTXO set:**
  - Stored in a separate leveldb (`LoadUtxoDB`, `internal/blockchain/utxobackend.go:326`) behind `UtxoCache` (default max 150 MiB, `config.go:51`).
  - `PutUtxos` writes the entries and the `UtxoSetState{lastFlushHeight, lastFlushHash}` marker atomically (`internal/blockchain/utxobackend.go:809-830`).
- **Indexers:**
  - `TxIndex` and `ExistsAddrIndex` are updated asynchronously through `IndexSubscriber` (`internal/blockchain/indexers/indexsubscriber.go:190-201, 248`).
  - Strict height ordering is enforced (`internal/blockchain/indexers/common.go:646`).
- **DB versioning:** schema v14 (`internal/blockchain/chainio.go:29`); upgrades v5→v14 in `internal/blockchain/upgrade.go:6242`; versions older than v5 are refused (`:6202`).

### Application interfaces (`internal/rpcserver`, `rpcadaptors.go`)
- **Routing and interfaces:**
  - HTTP `/` and websocket `/ws` (`internal/rpcserver/rpcserver.go:5916-5930`).
  - Adapters over chain, sync, mempool and mixpool are defined in `internal/rpcserver/interface.go` (`Chain :252`, `SyncManager :178`, `TxMempooler :630`, `MixPooler :662`) and implemented in `rpcadaptors.go`.
- **Authentication:**
  - HMAC-SHA256 of the Basic-auth header compared with `subtle.ConstantTimeCompare` (`internal/rpcserver/rpcserver.go:5481-5486`).
  - With no credentials, `checkAuth` passes (`:5521`); this is intended for client-cert TLS (`server.go:3694`).
  - The limited-user allowlist `rpcLimited` (`:341`) is enforced on HTTP (`:5624`) and websocket (`internal/rpcserver/rpcwebsocket.go:1512, 1721`). It includes the state-affecting calls `sendrawtransaction`, `submitblock`, `sendrawmixmessage` and `regentemplate`.
- **Limits:**
  - max clients 10, request body 8 MiB (`internal/rpcserver/rpcserver.go:5434, 5678`);
  - max websockets 25 (`internal/rpcserver/rpcwebsocket.go:113`);
  - pre-auth ws read limit 4 KiB, then 16 MiB;
  - per-ws concurrency semaphore (`:1549`).
- **Input validation:**
  - `parseCmd` (`internal/rpcserver/rpcserver.go:5571`, via `dcrjson`) produces typed commands; handlers then decode hex and wire data.
  - `handleSendRawTransaction` (`:4399`) calls `ProcessTransaction` with orphans disallowed.
  - `handleSubmitBlock` (`:4571`) calls `SyncMgr.SubmitBlock`.
  - `handleSendRawMixMessage` (`:4339`) calls `AcceptMixMessage`.

### Mixing (`mixing/mixpool`, `mixing/mixclient`)
- **Path:** `onMixMessage` (`server.go:1754`) → `netsync.OnMixMsg` (`internal/netsync/manager.go:1046`, which uses a rejected filter) → `mixpool.AcceptMessage` (`mixing/mixpool/mixpool.go:1155`). Signature verification is at `:1214`.
- **PR checks:** UTXO ownership proof and fee, through `checkAcceptPR` (`:1388`).
- **KE checks:**
  - epoch window: not early by more than 5s, not late by more than 20 min, and a whole minute (`checkAcceptKE` `:1669-1690`);
  - no distinct KE for the same identity and session (`acceptKE` `:1697-1721`).
- **Bounds:** at most 7·2·MaxPeers messages per identity (`:49, :1333`); orphans ≤ 250 (`limitNumOrphans` `:1067`).
- **Chain/mempool coupling:**
  - `mixpoolChain.FetchUtxoEntry` treats mempool-spent outpoints as missing (`server.go:3752`);
  - the mempool rejects non-mix spends of PR UTXOs (`internal/mempool/mempool.go:1549-1553`) and spends by misbehaving mix participants (`:1557-1561`);
  - `MisbehavingBlock` suppresses winning-ticket notifications (`server.go:2954-2957`).

### Protocol configuration and agenda activation (`chaincfg/`, `internal/blockchain/agendas.go`, `thresholdstate.go`)
- **Params:**
  - `chaincfg.Params` (`chaincfg/params.go:240`) carries `Deployments` (with `ForcedChoiceID`, `:185-197`) and the RuleChangeActivation* fields.
  - Testnet3 forces SDiff and LNFeatures (`chaincfg/testnetparams.go:180, 209`); simnet forces all agendas.
- **Agenda model:**
  - `consensusAgenda` (`internal/blockchain/agendas.go:401`) has an optional `forcedState`, `historicalState`, cached `activeAnchor` and `deployment`.
  - It is built and validated by `makeAgendas` (`:449`), which forbids forced choices on mainnet (`:492`).
- **Default required agendas** (`requiredAgendaIDs`, `internal/blockchain/agendas.go:35-49`): any of the 13 consensus IDs missing from params is forced Defined on mainnet and Active elsewhere.
- **Historical agendas:**
  - `makeHistoricalAgendas` (`internal/blockchain/agendas.go:66`) hard-codes the anchor (parent of the activation block) for MainNet and TestNet3.
  - Activation is resolved positionally: `isAgendaActivePositional` (`:620`).
- **Resolution order** (`isAgendaActive`, `:717`):
  1. forced state
  2. historical anchor (inactive below the anchor height, `:640`)
  3. cached anchor ancestor (`:652`)
  4. tally via `agendaState` (`internal/blockchain/thresholdstate.go:441`) → `nextThresholdState` (`:183`). The tally is cached per window, and moving from Defined to Started requires the stake-version gate (`:270`) and the PoW-version gate (`:277`).
- **Exported queries:** `Is*AgendaActive(prevHash)` wraps resolution with `CanValidate` and the chain lock (`isAgendaActiveByHash`, `:772`). Both consensus and mempool pass the *parent* or tip, so they evaluate rules for the next block.
- **Script flags:**
  - consensus: `consensusScriptVerifyFlags(node)` uses `node.parent` (`internal/blockchain/validate.go:4227-4251`);
  - policy: `standardScriptVerifyFlags` uses the tip (`server.go:3550-3574`) plus `BaseStandardVerifyFlags` (`internal/mempool/policy.go:67`).
- **Network-specific exceptions:** SBSS violation tables (`internal/blockchain/validate.go:149-164, 3368`), the testnet3 difficulty checkpoint (`:1311-1318`), and the DCP0005 whitelist (`:998`).

### Script engine (`txscript/`)
- `NewEngine` (`txscript/engine.go:697`), `Execute` (`:589`) and `Step` (`:479`).
- Limits: `MaxStackSize` 1024 and `MaxScriptSize` 16384 (`txscript/engine.go:64,67`); `MaxOpsPerScript` 255 and `MaxScriptElementSize` 2048 (`txscript/script.go:18-20`).
- Agenda-gated opcodes use flags `ScriptVerifySHA256` and `ScriptVerifyTreasury` (`txscript/engine.go:49-58`; checks at `txscript/opcode.go:2375, 2931-2962`).
- Parallel validation: `txValidator`/`ValidateTransactionScripts`/`checkBlockScripts` (`internal/blockchain/scriptval.go:37-224`).
- Signature cache: `SigCache` (`txscript/sigcache.go:63-181`), shared by the mempool and the chain.

### Startup (`dcrd.go`, `config.go`, `blockdb.go`)
- **Sequence:**
  1. `limits.SetLimits` (`dcrd.go:268`)
  2. `loadConfig` (`:43`)
  3. `loadBlockDB` (`:172`; `blockdb.go:103`)
  4. `LoadUtxoDB` (`:190`)
  5. index drops (`:211-236`)
  6. `newServer` (`:240`)
  7. `svr.Run` (`:261`; `server.go:3422`)
- **Config validation:** one network only (`config.go:796-816`), registered DB driver (`:922`), RPC auth rules (`:1021-1055`), `--notls` localhost-only (`:1191-1220`), no `AllowUnsyncedMining` on mainnet (`:1159`).

### Where responsibility transfers
```
socket ─peer.inHandler─► wire.ReadMessageN ─► peer.processInboundMessage ─► serverPeer.OnX (server.go)
   ├─ block ─► netsync.OnBlock ─► chain.ProcessBlock ─► notifications ─► server.handleBlockchainNotification
   │                                                   ├─► mempool (remove/re-add), bg template, RPC ntfns
   │                                                   └─► indexSubscriber.Notify (async)
   ├─ tx ────► netsync.OnTx ─► mempool.ProcessTransaction ─► server.AnnounceNewTransactions ─► relay
   ├─ headers► netsync.OnHeaders ─► chain.ProcessBlockHeader
   └─ mix ───► netsync.OnMixMsg ─► mixpool.AcceptMessage
RPC ─► rpcserver ─► adaptors (rpcadaptors.go) ─► netsync/chain/mempool/mixpool
mining ─► mining.TxSource (mempool) + chain queries ─► CheckConnectBlockTemplate ─► netsync.ProcessBlock
```

---

## End-to-End Processing Flows

### Flow A — Incoming block

| # | Stage | Function (file:line) | Data / branches |
|---|---|---|---|
| A1 | Receive | `Peer.inHandler` `peer/peer.go:1535` → `readMessage` `:963` (read deadline = `IdleTimeout`, `:965`) | Stall control msgs around handling (`:1573-1584`); any read error ⇒ disconnect (`:1556-1569`). getdata items carry 30 s deadlines (`:69, 1086-1125`). |
| A2 | Decode | `wire.ReadMessageN` `wire/message.go:362-498`; `MsgBlock.BtcDecode` `wire/msgblock.go:85-152` | Block `MaxPayloadLength` 1,310,720 B (`wire/msgblock.go:345-355`); per-tree tx count cap `MaxTxPerTxTree` (`:101-106,130-134`); returns msg + raw bytes. |
| A3 | Peer/server checks | `processInboundMessage` `peer/peer.go:1376-1379` → `serverPeer.OnBlock` `server.go:1356-1371` | Malformed msg ⇒ ban in `OnRead` (`server.go:1851-1859`). `OnBlock` builds `dcrutil.NewBlockFromBlockAndBytes`, marks inv known, calls netsync **synchronously** (back-pressure on that peer). |
| A4 | Sync layer | `SyncManager.OnBlock` `internal/netsync/manager.go:1216` → `processBlock` `:1185` | Unrequested ⇒ disconnect (`:1222-1231`). `ErrDuplicateBlock` ignored (`:1249-1251`); other errors only logged (`:1260-1270`) — **no ban for invalid blocks**. Rejected best header ⇒ reset `syncHeight` + refetch headers (`:1280-1292`). Success ⇒ mempool/mixpool pruning (`:1348-1357`), `fetchNextBlocks` (`:1369-1370`). RPC/miner blocks enter at `ProcessBlock` `:2197` without request gating. |
| A5a | Duplicate / known-invalid | `BlockChain.ProcessBlock` `internal/blockchain/process.go:445` (takes `processLock` `:448`, `chainLock` `:458`) | `index.HaveBlock` ⇒ `ErrDuplicateBlock` (`:452-456`); `checkKnownInvalidBlock` (`:463-468`). |
| A5b | Header first | `maybeAcceptBlockHeader` `internal/blockchain/process.go:159-221` (call `:482`) | `checkBlockHeaderSanity` (`internal/blockchain/validate.go:768`: PoW, timestamp ≤ adj+2h `:799-805`, voter bounds `:848-865`); parent known & not invalid (`internal/blockchain/process.go:183-195`); `checkBlockHeaderPositional` (`internal/blockchain/validate.go:1239`: median time `:1249-1255`, difficulty, height, checkpoint `:1305-1312`); `index.AddNode` (`internal/blockchain/process.go:208-210`). |
| A5c | Data preconditions | `checkBlockDataPreconditions` `internal/blockchain/validate.go:1978-2049` (call `internal/blockchain/process.go:496`) | Wire size + merkle root(s) using positional HeaderCommitments state; returns `dataCommitProven` (false ⇒ either merkle variant tolerated provisionally). |
| A5d | Data sanity | `checkBlockDataSanity` `internal/blockchain/validate.go:881-966` (call `internal/blockchain/process.go:505`) | PoS ticket prices, `header.Size` match (`:910`), per-tx `standalone.CheckTransactionSanity`, in-block dup tx hashes (`:952-961`). Failure marks block invalid **only if** `dataCommitProven` (`internal/blockchain/process.go:510-513`). |
| A5e | Fast-add decision | `internal/blockchain/process.go:520-524`, `isAssumeValidAncestor` `:116-140` | Sets `BFFastAdd` + `statusValidated` for bulk import / assume-valid ancestors (skips lottery, sbits, some stake checks; see Invariants I2/I9). |
| A5f | Store data | `maybeAcceptBlockData` `internal/blockchain/process.go:287-338` (call `:537`) | `checkBlockDataPositional` (expiry); `pruneChainIfNeeded`; **block written to ffldb** `dbMaybeStoreBlock` (`:321-323`); `statusDataStored`; `index.AcceptBlockData` returns newly linked descendants (`internal/blockchain/blockindex.go:1309-1373`). `flushBlockIndex` (`internal/blockchain/process.go:552`). |
| A5g | Context checks | `maybeAcceptBlocks` `internal/blockchain/process.go:356-411` → `checkBlockContext` `internal/blockchain/validate.go:2104` | Header context (`:1478`), merkle context unless cached (`:2134-2143`), coinbase height, vote parent commitments, sigops ≤ 5000 (`:2378-2408`), consensus max size (`:2413-2422`). RuleError (≠ `ErrBadMerkleRoot`) ⇒ `MarkBlockFailedValidation` (`internal/blockchain/process.go:367-372`). `NTNewTipBlockChecked` sent **with chainLock held** (`:403-405`). |
| A6 | Chain selection & connect | `FindBestChainCandidate` (`internal/blockchain/process.go:589`) → `reorganizeChain` `internal/blockchain/chain.go:1269-1397` → `reorganizeChainInternal` `:1039-1250` | Detach: spend journal `dbFetchSpendJournalEntry` (`:1094-1097`) → `view.disconnectBlock` → `b.disconnectBlock`. Attach: if not validated, `checkBlockContext` (`:1208`) + `checkConnectBlock` (`:1223`; `internal/blockchain/validate.go:4370-4640`: dup txs, `checkTransactionsAndConnect` `:3994` per tree, `tspendChecks`, sequence locks, scripts, GCS commitment, `view.SetBestHash` `:4637`). Rule error ⇒ mark failed, retry next candidate (`internal/blockchain/chain.go:1226-1229, 1326-1338`). |
| A7 | State/index updates | `connectBlock` `internal/blockchain/chain.go:586-800` | Asserts tip linkage & stxo count (`:590-609`); `fetchStakeNode`; `bestChain.SetTip` (`:735`); state snapshot swap; chainLock **released** for `NTBlockConnected`/`NTNewTickets` (`:763-769`); `NTBlockAccepted` per accepted node after reorg (`internal/blockchain/process.go:646-665`). Server reacts in `handleBlockchainNotification` (`server.go:2872`; connect `:3002-3132`). |
| A8 | Persistence | `connectBlock` single `db.Update` (`internal/blockchain/chain.go:661-708`); `utxoCache.Commit` (`:717`) + `MaybeFlush` (`:728`) | One ffldb tx: best state, spend journal, stake `WriteConnectedBestNode`, treasury balance/tspend, GCS filter, header commitments. UTXO flush (`internal/blockchain/utxocache.go:681-786`) **first flushes the block DB** (`:713`) then `PutUtxos` with consistency marker (`internal/blockchain/utxobackend.go:809-830`). Indexers notified asynchronously (`internal/blockchain/indexers/indexsubscriber.go:190-201`). |

Recovery path (startup):
1. `ffldb.reconcileDB` (`database/ffldb/reconcile.go:53-115`) truncates flat files that run past the write cursor.
2. `initChainState` (`internal/blockchain/chainio.go:1577`, tip `:1686-1714`) loads the tip.
3. `UtxoCache.Initialize` (`internal/blockchain/utxocache.go:815-1041`) disconnects or replays blocks between the UTXO marker and the tip.

### Flow B — Incoming transaction

| # | Stage | Function (file:line) | Continue/stop conditions |
|---|---|---|---|
| B1 | Announce / request | `serverPeer.OnInv` `server.go:1377` → `SyncManager.OnInv` `internal/netsync/manager.go:1880` (tx `:1914-1937`), `needTx` `:1833-1850` | Empty inv ⇒ ban (`server.go:1379-1383`); not current ⇒ skip; known/rejected/recently-confirmed ⇒ skip; `requestedTxns` cap `MaxInvPerMsg` (`internal/netsync/manager.go:82`). |
| B2 | Receive | `inHandler` → `processInboundMessage` `peer/peer.go:1371-1373` → `serverPeer.OnTx` `server.go:1326-1356` | blocksonly ⇒ drop (`:1327-1331`); mark known inv; synchronous `syncManager.OnTx` (`:1348`). |
| B3 | Decode | `wire.ReadMessageN`; `MsgTx.BtcDecode` `wire/msgtx.go:792-899` | Tx payload ≤ 1.25 MB; ser-type in high 16 bits (`:803-804`); in/out count caps (`:106-116, 641, 669`); negative outpoint tree ⇒ `ErrNegativeTxTree` (`:1193`, commit 3dde63a). |
| B4 | Sync layer | `SyncManager.OnTx` `internal/netsync/manager.go:968-1022` | Unsolicited txs accepted; hash in `rejectedTxns` APBF ⇒ ignore (`:986-990`); calls `ProcessTransaction(tx, allowOrphans=MaxOrphanTxs>0, allowHighFees=true, tag=peerID)` (`:994-996`); **any** error ⇒ add to `rejectedTxns` (`:1009`). RPC path: `internal/rpcserver/rpcserver.go:4421` (no orphans). |
| B5 | Tx checks | `ProcessTransaction` `internal/mempool/mempool.go:2360` (agenda flags before lock `:2363`; `mp.mtx` `:2369`) → `maybeAcceptTransaction` `:1400` | Duplicate (`:1409-1414`); `blockchain.CheckTransaction` (`:1419`; `internal/blockchain/validate.go:644-650` → `standalone.CheckTransactionSanity` `blockchain/standalone/tx.go:147` + `checkTransactionContext` `internal/blockchain/validate.go:412`); `DetermineTxType` (`:1443`); reject coinbase/treasurybase, expired, pre-SVH votes/TSpends (`:1452-1509`); standardness `checkTransactionStandard` (`internal/mempool/policy.go:298-389`); ticket ≥ next sdiff (`:1534-1545`); mix hooks (`:1549-1561`). |
| B6 | Pool conflicts & inputs | `checkPoolDoubleSpend` `:1032`; vote rules `checkVoteDoubleSpend` `:1072`, `checkVoteHeightPolicy` `:1111`, `checkVoteBlock` `:1143`; UTXO fetch → orphan detection | Missing parents ⇒ `ErrOrphan` (RPC) or `maybeAddOrphan` (`:637-646`, `:2403-2413`). Sequence locks (`:1724-1734`); `CheckTransactionInputs` ⇒ fee (`:1746-1752`); `checkInputsStandard`; sigops ≤ MaxSigOpsPerTx (`:1773-1782`); min fee (`:1784-1821`); max fee unless `allowHighFees` (`:1826-1835`); scripts with **standard** flags (`:1839-1847`); `checkVoteTicket` winning-ticket check (`:1857-1861`, `:1168`); TSpend policy (`:1864-1932`). |
| B7 | Index/state update | `addTransaction` `internal/mempool/mempool.go:992-1024`; then `voteTrack.AddVote` `:1975`, `tspends` `:1980` | Ticket spending mempool output ⇒ stage pool (`:1949-1954`). `pool`, `outpoints`, `miningView.AddTransaction`, `ExistsAddrIndex.AddUnconfirmedTx`, fee estimator. `OnVoteReceived` → bg template (`:998-1000`, `server.go:4099-4103`). `processOrphans` (`:2125-2208`). |
| B8 | Relay | `server.AnnounceNewTransactions` `server.go:1992-2002` → `RelayInventory` `:2772` | Peers with relay disabled skipped; tx cached in `recentlyAdvertisedTxns` (`:2097-2119`). |
| B9 | Template selection | `NewBlockTemplate` `internal/mining/mining.go:1170` | Snapshot `MiningView()` (`:1307`); prefilter (`:1340-1437`: finality, vote parent match, inputs available); selection loop (`:1494-1838`: TSpend on TVI, TAdd/ticket caps, size/sigops, vote's ticket ∈ `NextWinningTickets`, bundle re-validation `:1752,1762`); stake tree (`:1911-2059`); fees → coinbase (`:2102-2157`); `CheckConnectBlockTemplate` (`:2338`). |

Mempool reaction to chain events (`server.go:3002-3212`):
- On connect, for each tx: `RemoveTransaction`, `MaybeAcceptDependents`, `RemoveDoubleSpends`, `RemoveOrphan`, `ProcessOrphans`, `TransactionConfirmed`.
- A disapproved parent's regular txs are re-added with `MaybeAcceptTransactions` (`internal/mempool/mempool.go:2082-2117`).
- On disconnect, block txs are re-added.
- `PruneStakeTx` and `PruneExpiredTx` run from netsync for network blocks (`internal/netsync/manager.go:1348-1352, 2209-2213`).

---

## Correctness Invariants

Status legend: **Located** means the implementation was found and read.
**VERIFICATION PENDING** means no implementation was found, or it is only partly confirmed.

| ID | Invariant | Responsible | Implemented / checked at | Dependents | Assumptions |
|---|---|---|---|---|---|
| I1 | **Layered block validation.** No block is connected without its sanity, positional, context and connect checks, except where BFFastAdd applies. | `internal/blockchain` | `internal/blockchain/process.go:445-665`; `internal/blockchain/chain.go:1205-1233`. Context results cached in `recentContextChecks` (`internal/blockchain/validate.go:2112`); merkle results in `recentMerkleChecks` (`internal/blockchain/chain.go:209`). Invalid status is set only when `dataCommitProven` or the error is not `ErrBadMerkleRoot` (`internal/blockchain/process.go:510-513, 367-372`; `internal/blockchain/chain.go:1209-1212`). | netsync, RPC submitblock, mining, indexers | `maybeAcceptBlockData` is only called after sanity checks (`internal/blockchain/process.go:281-283`). Positional agenda facts are immutable history (`internal/blockchain/validate.go:1960-1975`). |
| I2 | **Mempool validity matches block validity at admission.** Consensus helpers are shared; policy adds stricter rules. | mempool + chain | Shared: `CheckTransaction`, `CheckTransactionInputs` (`internal/blockchain/validate.go:3394`), `ValidateTransactionScripts`, `CalcSequenceLock` (`internal/mempool/mempool.go:1419, 1724-1752, 1839`). Policy only: `internal/mempool/policy.go:110-389`, vote and TSpend rules. Re-validated per bundle at template time (`internal/mining/mining.go:1752-1770`) and by the final `CheckConnectBlockTemplate`. | mining, relay | Chain state can change after admission; pruning and template re-validation are what maintain the invariant. The mempool's chain callbacks are not one atomic chain view (separate `chainLock` acquisitions). |
| I3 | **Amount accounting.** Outputs and inputs stay in range with no overflow; fees ≥ 0; coinbase ≤ subsidy + scaled fees; stake tree ≤ vote subsidy × voters. | standalone + chain | `blockchain/standalone/tx.go:170-200`; input sums with `addSigned` (`internal/blockchain/validate.go:3514-3530, 3730-3750`); block fees (`:4072`); coinbase (`:4149`, scaling `:4111`); stake tree (`:4199-4214`); `internal/blockchain/checkedmath.go:9-23`. | mining fee scaling (`internal/mining/mining.go:2138-2157`) | The unchecked `totalAtomOut +=` (`internal/blockchain/validate.go:3741-3744`) relies on the earlier sanity check bounding each output. |
| I4 | **No duplicate state application.** | chain | Block: `ErrDuplicateBlock` (`internal/blockchain/process.go:452-456`). Tx within a block: `internal/blockchain/validate.go:952-961`. Duplicate inputs: `blockchain/standalone/tx.go:223`. Spent or missing output: `ErrMissingTxOut` (`internal/blockchain/validate.go:3569`); same-block spends tracked via `inFlightTx` (`internal/blockchain/utxoviewpoint.go:344-406`). **BIP30-style check disabled** (`checkForDuplicateHashes = false`, `internal/blockchain/validate.go:100`; early return `:2548-2550`). | UTXO cache, indexers | Cross-block txid uniqueness relies on coinbase and treasurybase height commitments (`internal/blockchain/validate.go:1569, 1662`). |
| I5 | **Deterministic state transitions.** Validity depends only on the parent state, the block, and agenda state derived from the parent. | chain | Agenda queries take the parent (`internal/blockchain/agendas.go:717`); script flags use `node.parent` (`internal/blockchain/validate.go:4227-4251`). | all peers (consensus) | Future-timestamp check uses local adjusted time (`internal/blockchain/validate.go:799-805`), so it is node-local and header-only. Tie-break in chain selection is local (`internal/blockchain/blockindex.go:446`), so it is not consensus. `calcVoterVersionInterval` iterates a map (`internal/blockchain/stakeversion.go:212-216`): deterministic only if at most one version can reach 3/4 (VERIFICATION PENDING at the params level). |
| I6 | **Consistent UTXO updates.** The view is at the parent; the commit is atomic with best state from recovery's point of view. | chain + UtxoCache | `view.BestHash == parent` assertion (`internal/blockchain/validate.go:4373`); `SetBestHash` (`:4637`); `UtxoCache.Commit` (`internal/blockchain/utxocache.go:548-638`); stxo count assertions (`internal/blockchain/chain.go:605-609`, `internal/blockchain/utxoviewpoint.go:728-732`); spend journal written in the same ffldb tx as best state (`internal/blockchain/chain.go:661-675`). | mempool `FetchUtxoView`, mining | Block DB is flushed before the UTXO DB (`internal/blockchain/utxocache.go:713-727`). |
| I7 | **Reorg consistency.** Disconnect exactly reverses connect. | chain + stake | Spend-journal stxos (`internal/blockchain/chain.go:1094-1108`; `internal/blockchain/utxoviewpoint.go:429`); journal entry deleted only after a forced UTXO flush (`internal/blockchain/chain.go:893-911`); linkage panics (`:1071-1075, 1166-1170`); stake undo data (`blockchain/stake/tickets.go:588-698, 797-903`); `WriteDisconnectedBestNode` (`:1021-1040`). | mempool re-add (`server.go:3144-3212`), indexers | Stake undo in the DB is keyed **by height**, so it is valid only for main-chain walks (`internal/blockchain/stakenode.go:142-167`). A failed attach leaves a partial reorg, which is then resolved by retry (`internal/blockchain/chain.go:1326-1338`). |
| I8 | **Stake and vote accounting.** Winners match the lottery; vote count and majority match the header; revocations match missed/expired tickets; payouts follow the commitments. | chain + stake | PoolSize/FinalState (`internal/blockchain/validate.go:1543, 1553`); winners (`:1745-1773`); voter count (`:2326`); approval bit (`:2335-2344`); vote subsidy (`:3105`, `:3908`); commitments (`:2908`, `:2783`); revocation eligibility and auto-revocation completeness (`:1812-1847`); `connectNode` backstops (`blockchain/stake/tickets.go:539-545, 643-652`); maturity (`internal/blockchain/validate.go:3151, 3256-3271`). | mempool vote rules, mining stake tree | **Under BFFastAdd the lottery, sbits and auto-revocation completeness checks are skipped** (`internal/blockchain/validate.go:1497, 2457`); only `connectNode` membership backstops remain (trust in assume-valid). |
| I9 | **Treasury.** Balance ≥ spend; spend ≤ policy cap; each TSpend at most once per chain, with sufficient votes, inside its window. | chain | `tspendChecks` (`internal/blockchain/validate.go:4263`, called unconditionally at `:4489`); `checkTSpendsExpenditure` (`internal/blockchain/treasury.go:899-938`); `checkTSpendExists` (`:943`); `checkTSpendHasVotes` (`:1114`); overflow-checked sum (`internal/blockchain/validate.go:4297-4303`). | mining TSpend selection (`internal/mining/mining.go:1561-1606`) | TSpend `ValueIn` was validated earlier (`internal/blockchain/validate.go:3327, 3499`). `calculateTreasuryBalance` returns 0 on any DB error (`internal/blockchain/treasury.go:381-392`): see Open Questions. |
| I10 | **Predictable protocol activation.** Threshold state is constant within a window, cached per window, and identical across validation, mining and mempool. | chain (`agendas.go`, `thresholdstate.go`) | `nextThresholdState` (`internal/blockchain/thresholdstate.go:183-235`); positional/historical/anchor resolution (`internal/blockchain/agendas.go:620-717`); `makeAgendas` construction invariants (`:449-588`); chain-lock requirement for cache mutation (`:618`, `internal/blockchain/thresholdstate.go:182`). | validation (`internal/blockchain/validate.go:348`), mining (`internal/mining/mining.go:1205-1222`), mempool (`server.go:4109-4124`), RPC (`getvoteinfo` etc.), DB upgrade (`internal/blockchain/upgrade.go:2648`) | Hard-coded anchor hashes are only height-checked by tests (`internal/blockchain/agendas_test.go:256`): VERIFICATION PENDING against real chain data. RPC threshold queries bypass the positional short-circuit (`internal/blockchain/thresholdstate.go:441, 467, 552`). |
| I11 | **Bounded processing and resource usage.** | wire, peer, netsync, chain, mempool, rpc, mixpool | Wire: 32 MiB per message and per-type caps (`wire/message.go:411, 437-446`). Peer output: 40 MiB (`peer/peer.go:1848-1857`). getdata quotas (`server.go:120-121, 1440-1457`). netsync request maps and in-flight limits (`internal/netsync/manager.go:38, 78-82, 96`). Block: 5000 sigops and size (`internal/blockchain/validate.go:2378-2422`). Orphans: 100 × 100 KB with TTL (`internal/mempool/mempool.go:550-646`). Vote tracker pruning (`:385-398`). RPC limits (`internal/rpcserver/rpcserver.go:5434, 5678`; `internal/rpcserver/rpcwebsocket.go:113, 1549`). Mixpool orphans and per-identity caps (`mixing/mixpool/mixpool.go:49, 1067, 1333`). Chain LRUs (`internal/blockchain/chain.go:2118-2133`). | everything | **Main mempool has no size cap** (VERIFICATION PENDING whether any bound exists beyond fees and pruning). Total mixpool PR/session count: VERIFICATION PENDING (expiry only, `mixing/mixpool/mixpool.go:570`). |
| I12 | **Persistence and recovery.** On restart, state equals some consistent prefix of the processed chain. | ffldb + UtxoCache + chainio | ffldb tx atomicity and flat-file rollback (`database/ffldb/db.go:1697-1729, 1948`); `reconcileDB` (`database/ffldb/reconcile.go:53-115`); tip reload (`internal/blockchain/chainio.go:1686-1714`); UTXO replay (`internal/blockchain/utxocache.go:853-1037`); clean-shutdown flush (`internal/blockchain/utxocache.go:1055-1063`, `server.go:3505`); indexer tip atomic with index data (`internal/blockchain/indexers/txindex.go:544`, `internal/blockchain/indexers/common.go:684`). | node availability | Invariant: block DB ≥ UTXO DB. Every spend-journal entry between `lastFlushHash` and the fork still exists. Durability of the previous flat file at rollover is unverified (Open Questions). |
| I13 | **Mixpool acceptance.** Messages carry valid signatures; PR UTXOs are owned and unspent; KE timing is enforced. | mixpool | `mixing/mixpool/mixpool.go:1214, 1388-1513, 1669-1724`. | mempool hooks, voting suppression | UTXO check only runs when a `UtxoFetcher` is configured (`mixing/mixpool/mixpool.go:293`). Reconsidered orphans skip some checks (Open Questions). |

---

## Recent Changes and Integration Areas

Window: 885ea18 (v2.1.6 release notes) to 17dd9f4, 69 commits. No change below
is claimed to introduce a defect.

| Commit(s) | Component | Summary | Related interfaces / dependents | Worth checking |
|---|---|---|---|---|
| 024899e | `internal/blockchain` process/validate | Headers are accepted (full sanity, positional, PoW) **before** data checks. `checkBlockSanity` is split; new `checkBlockDataSanity` (`internal/blockchain/validate.go:881`). Failed-marking moved into `ProcessBlock`. A header with bad data now remains indexed and is marked invalid. | netsync `:1185`, `:1620`; `cmd/addblock/import.go:134`; `rpcadaptors.go:406, 550` (`CheckBlockSanity`) | `TestProcessLogic` invalid-header and bad-block cases |
| ddd3b6d, b56e78f | `internal/blockchain` | Early data-commitment preconditions with the `merkleRootVariant` enum; `dataCommitProven` gates invalid-marking. `recentMerkleChecks` LRU of size 25 (`internal/blockchain/chain.go:209`), cleared on invalidate/reconsider (`internal/blockchain/process.go:709, 837, 889`). | `CheckConnectBlockTemplate` (`internal/blockchain/validate.go:4680`); mining `:933, 2338` | Mutated-data case `bfb` in `process_test.go`; simnet/regnet non-definitive path; cache across reorg |
| ead8ba7 | validate | Max-size and merkle checks now run under BFFastAdd too | assume-valid path (`internal/blockchain/process.go:520-524`) | No direct fast-add negative test |
| cd07eee, 9dc1c4f, 82d927b, e0f917d, 05c31a8, 476d2da, ea2689d, 9a12399, dbb1cbc (+ tests 43b67ce, 88e6a8d, 5cf38f6, 20bc751) | agendas / threshold state | Agenda vs deployment separation (`consensusAgenda`). Consolidated `isAgendaActive`. Stake-diff and max-size queries now return errors. `ThresholdStateTuple.ChoiceID string` (**exported API change**). Default required agendas; active anchors; hard-coded historical anchors; positional queries. | `calcNextRequiredStakeDifficulty` (`internal/blockchain/difficulty.go:860`) and `maxBlockSize` (`internal/blockchain/chain.go:1615`) callers; RPC `NextThresholdState`/`StateLastChangedHeight` (`internal/rpcserver/interface.go:394, 400`); DB upgrade `internal/blockchain/upgrade.go:2648` | Anchor hash/height provenance; side chains below anchors; RPC vs validation agreement; `agendas_test.go` |
| bf449ef | chain locking | Agenda queries mutate caches, so they need the write lock. `FetchUtxoView` changed from RLock to Lock; `MaxTreasuryExpenditure` takes the lock. | mempool `FetchUtxoView` (`server.go:~4078`); RPC/mining | `-race` runs; mempool/RPC contention |
| 6444f05, a9b79d1, 6816d9d, 023c42f, d54099a, 5a6f22f, 93544c3, f367cdf, 7388fcd, cd33677, e8f47dc, 8496124, b443b0b | mempool | Separate `voteTracker`; reject future votes (>5), votes on unknown blocks or wrong heights, and ineligible tickets (new `Config.WinningTicketsByHash` → `LotteryDataForBlock`, takes exclusive `chainLock`); prune old vote metadata (`heightDiffToPruneVotes=10`). | `mining.TxSource` docs (`internal/mining/interface.go:43-55`); `internal/mining/mining.go:348, 377, 1278, 1333`; `internal/mining/bgblktmplgenerator.go:791`; `server.go:1169, 1282, 4082` | Lock order mempool → tracker → chainLock; prune threshold vs `MaxVoteAge`; tests `internal/mempool/mempool_test.go:1956, 2078, 2420, 2573` |
| 3c8a173 | mempool ↔ mixpool | `MisbehavingMixSpend` hook → `Observer.MisbehavingTx` | `server.go:4132`; `mixing/mixpool/observer.go:462` | No acceptance-path test sets the hook |
| 70485d9, ffb42ad, 3ce5cc4, 38811b0, 874f677, 6b65e0d, 83beff1, 6aa6b46, 1d78ba4, c658232 | mixing | Linear `activeInEpoch`; distinct-KE rejection; late-KE (20 min); per-identity cap; whole-minute epoch; KE `SeenPRs` cap removed (now only wire's 512); SR dimension check before pad recovery | `netsync.OnMixMsg`; mixclient epoch truncation | Few new tests (identity cap, late KE, duplicate KE, >64-peer KE) |
| 17dd9f4 | netsync | `startInitialHeaderSync` sets `syncHeight = max(peer.LastBlock, bestHeaderHeight)` unconditionally (`internal/netsync/manager.go:682`) | `SyncHeight()` consumers: `:1121, 1316, 1427, 1723`; `rpcadaptors.go:418` | No test (only `TestMaybeUpdateSyncHeight`) |
| 3dde63a, 6f6cf21, efa17f6 | wire | Negative outpoint tree rejected (`ErrNegativeTxTree`). New error code inserted before `ErrInvalidMsg`, which **shifts later iota values**. wire v1.8.0 prepared. | `readTxInPrefix`/`writeTxInPrefix`; `MsgMixPairReq`; `MsgTx.TxHash` panics on serialize error (`wire/msgtx.go:418-429`) | In-memory txs with a negative tree; consumers comparing numeric error codes |
| 5dd4ca5, 036b709, 7e3dd85 | stake | `CreateRevocationFromTicket` allows zero amounts (`blockchain/stake/staketx.go:1408-1411`); treap `getByIndex` bound fixed; stake v5.0.3 prepared | mining auto-revocations (`internal/mining/mining.go:989`); RPC (`internal/rpcserver/rpcserver.go:1148`) | Zero-commitment auto-revocation through the template |
| b9634e0, a0cdd43, 26ea49a, bf449ef | chain | Shadowing fixes (`internal/blockchain/chainio.go:1696` `newRulesStartTime` now assigned; spend-journal migration `blockBytes`); removal of `ErrStakeFees` and `CheckProofOfStake` | startup `initChainState` | Behaviour of the now-reachable `flushBlockIndex` at `internal/blockchain/chainio.go:1784` |
| ae4c581, efc7e3d | standalone / primitives subsidy | Avoid caching duplicate subsidy intervals under a race | subsidy cache users (validation, mining) | No test added |
| 2bd17b2, 70ba63e, 90accde, 855dc8b, 1995f4f, b6c6757 | tests | New fullblocktests (revoke input, ticket input script/version) and validate tests (auto-revocation index, immature spend, invalid ticket input) | `fullblocks_test.go` error mapping | — |
| 9b7ab54 (just before window) | validate | Network-aware SBSS violation tables (testnet3 height 1980161) | `internal/blockchain/validate.go:149-164, 3368, 3578, 4449-4453` | — |
| 025e7c6, ffecb8e, 9ae5864, 495124d, 7080c11 | build | Go 1.26/1.27 CI; `debug.go` `go1.26` GODEBUG defaults; lint v2.13.1 | `go.mod` still declares `go 1.25.0` | Toolchain/GODEBUG behaviour differs between 1.25 and 1.26 builds |

---

## Input and State Assumptions

| Entry point | Assumption | Checked at | Inherited from |
|---|---|---|---|
| `wire.ReadMessageN` (`wire/message.go:362`) | `pver` is the negotiated version (local max before `version` arrives) | — | peer handshake (`peer/peer.go:967`) |
| Peer `OnX` handlers (`server.go`) | Message is structurally bounded; handlers are serialized per peer but share server state | Semantic checks in the handler (empty inv/getdata ⇒ ban `:1380, 1419`) | wire decode |
| `OnVersion` (`server.go:973`) | Runs **before** the peer's own too-old check; uses only Addr, Inbound, ID and ProtocolVersion | Server re-checks the minimum version (`:999`) | contract at `peer/peer.go:2306-2324` |
| `SyncManager.OnBlock` (`internal/netsync/manager.go:1216`) | Block was requested from this peer; synchronous call provides back-pressure | `isRequestedBlockFromPeer` (`:1222-1231`) | server `OnBlock` |
| `SyncManager.OnHeaders` (`:1506`) | Headers are contiguous by hash and height+1 | `:1596-1605` (chain re-checks the parent) | — |
| `BlockChain.ProcessBlock` (`internal/blockchain/process.go:445`) | Parent **header** is known (parent data not required; context checks wait until linked) | `ErrMissingParent` (`internal/blockchain/process.go:183-187`) | wire bounded size and tx counts |
| Timestamps | Header ≤ adjusted time + 2h; > median of the last 11 blocks; one-second precision | `internal/blockchain/validate.go:799-805, 1249-1255` | `MedianTimeSource` (peer time samples, `server.go:1045`) |
| `checkConnectBlock` (`internal/blockchain/validate.go:4370`) | View is at the parent; stake node of the parent is available | `:4373`; `fetchStakeNode` (`internal/blockchain/stakenode.go:97`) | reorg loop |
| `CheckTransactionInputs` (`internal/blockchain/validate.go:3394`) | Sanity already done; tx type determined; UTXO view includes earlier txs in the block | Comment `:3392-3393`; type preconditions panic (`:2668-2672`, `:3160-3163`) | `CheckTransaction` / block sanity |
| `checkVoteInputs` (`internal/blockchain/validate.go:3081`) | The vote's voted-on height matches the parent | Enforced only in `checkBlockContext` (`:2259`), not here | mempool relies on `checkVoteBlock` (`internal/mempool/mempool.go:1143`) |
| `stake.CheckSStx` (`blockchain/stake/staketx.go:713`) | Tx has at least one input | Not checked here (`2·0+1` passes alone) | `CheckTransactionSanity` (`blockchain/standalone/tx.go:147`) |
| `ProcessTransaction` (`internal/mempool/mempool.go:2360`) | Tx decoded by wire; mempool lock held throughout | All consensus and policy checks re-run | Agenda flags read **before** lock (`:2363`); best-state fields from separate snapshots (`server.go:4075-4124`) |
| `MaybeAcceptTransactions` (`internal/mempool/mempool.go:2082`) | Txs supplied in block order | Comment `:2072-2074` | server notification handler |
| `NewBlockTemplate` (`internal/mining/mining.go:1170`) | Single `best` snapshot for chain state; TxSource is concurrency-safe | Final gate `CheckConnectBlockTemplate` (`:2338`) | `MiningView` clone (`internal/mempool/mempool.go:2523-2530`) |
| `fetchStakeNode` (`internal/blockchain/stakenode.go:97`) | Tip always has a stake node; ancestors' block data present; DB undo data is main chain only | Comment `:122-124` | `connectBlock` stake writes |
| Agenda queries (`internal/blockchain/agendas.go:772`) | `CanValidate(prevNode)`; positional variant needs no ancestor block data | `:779`; `:601-605` | block index |
| `mixpool.AcceptMessage` (`mixing/mixpool/mixpool.go:1155`) | Hash precomputed; wire bounds applied; chain/UTXO view current | Signature, PR and KE semantics re-checked | `peer/peer.go:971`; `internal/rpcserver/rpcserver.go:4379` |
| RPC handlers | `parseCmd` produced a typed struct; auth/limited decision made upstream | `internal/rpcserver/rpcserver.go:5571, 5624` | — |
| ffldb open | leveldb metadata intact; write-cursor row well-formed | CRC (`database/ffldb/reconcile.go:34-49`); `deserializeWriteRow` indexes `[0:12]` without a length check (`:36-37`) | — |
| Config | Exactly one network; registered DB type; RPC auth combination valid | `config.go:796-816, 922, 1021-1055, 1191-1220` | — |

---

## Open Questions

Each item is unconfirmed. None is asserted to be a defect.

**Chain and synchronization**

1. **`NTBlockAccepted` payload** (`internal/blockchain/process.go:659-663`). Every accepted node's notification carries the *processed* `block`, while `ForkLen` is computed per node. Is this intended when one block links several descendants? Consumer: `server.go:2893-2999`.
2. **No penalty for invalid blocks.** netsync `OnBlock` neither bans nor disconnects a peer that delivers a rule-violating block (`internal/netsync/manager.go:1247-1295`). Does any other layer penalise it?
3. **`disconnectBlock` persists `node.workSum` with the parent's state** (`internal/blockchain/chain.go:861`). No non-test consumer of the persisted value was found (`internal/blockchain/chainio.go:1241`).
4. **Metadata cache vs crash.** ffldb metadata can lag in `dbCache` by up to 5 min or 100 MB (`database/ffldb/dbcache.go:22,27`). Recovery relies on `flushBlockDB` before every UTXO flush. Behaviour after losing cached metadata while flat files hold newer blocks: VERIFICATION PENDING.
5. **Flat-file durability at rollover.** The old block file is closed without `Sync` (`database/ffldb/blockio.go:434-437`) and `syncBlocks` syncs only the current file (`:603-620`). VERIFICATION PENDING against fsync semantics.
6. **Sync-height trust after header sync** (`internal/netsync/manager.go:760, 833`). The 17dd9f4 reset applies only before initial header sync completes. What happens with an inflated `LastBlock` after that?
7. **Bound on unlinked-children growth** (`internal/blockchain/blockindex.go:1368-1370`). This relies on netsync request gating, which RPC `submitblock` bypasses. The actual bound is unverified.
8. **Indexer catch-up after an unclean shutdown** (`internal/blockchain/indexers/indexsubscriber.go:248`). Unverified.

**Stake, treasury and agendas**

9. **Assume-valid trust scope.** Under BFFastAdd, lottery, sbits and auto-revocation completeness checks are skipped (`internal/blockchain/validate.go:1497, 1513, 2457`). Confirm this is the intended trust model.
10. **Hard-coded historical anchors** (`internal/blockchain/agendas.go:68-135`). Tests check only height alignment (`internal/blockchain/agendas_test.go:256`). Is there an offline hash check against real headers?
11. **Side chains forking below an anchor are unconditionally inactive** (`internal/blockchain/agendas.go:640`). Do the checkpoint and fork-rejection heights (`internal/blockchain/validate.go:1298-1312`) exceed all anchors?
12. **RPC threshold queries vs validation.** `NextThresholdState`, `GetVoteInfo` and `StateLastChangedHeight` use `agendaState` without the positional short-circuit (`internal/blockchain/thresholdstate.go:441, 467, 552`). Could they disagree with validation on side chains?
13. **`calculateTreasuryBalance` returns 0 on any DB error** (`internal/blockchain/treasury.go:381-392`), and that value could be persisted by `dbPutTreasuryBalance`.
14. **Spend-journal upgrade v2→v3 calls `IsTreasuryAgendaActive`** (`internal/blockchain/upgrade.go:2648`). That now goes through positional and anchor logic and needs `CanValidate`. Does it hold during the upgrade for every historical block?
15. **Determinism of `calcVoterVersionInterval` map iteration** (`internal/blockchain/stakeversion.go:212-216`). It needs a params-level guarantee that at most one version can reach the threshold.

**Mempool and mining**

16. **Mempool agenda flags span several tip snapshots** (`server.go:4109-4124`; `internal/mempool/mempool.go:1992-2011`). These are policy only; the impact at rule-change boundaries is not assessed.
17. **Stale `best` snapshot after `ForceHeadReorganization`.** `best` is not refreshed after the reorg (`internal/mining/mining.go:1200, 1271`; later uses at `:1462, 1629, 1723, 2009`). A cancelled background generation can still reorganize (`internal/mining/bgblktmplgenerator.go:708-738`).
18. **`OnVoteReceived` fires before `voteTrack.AddVote`** (`internal/mempool/mempool.go:999` vs `:1975`). The background generator may count votes before the new one is visible (`internal/mining/bgblktmplgenerator.go:790-792`).
19. **Relayed tx object vs pool copy.** The relayed object is the caller's original tx, while the pool stores a fraud-proof-corrected copy (`internal/mempool/mempool.go:1692-1694, 2394`; `server.go:546`).
20. **Outpoint trees 2–127 pass the wire layer** (`wire/msgtx.go:1189`). No explicit sanity rejection was found. Is that intended? They surface as orphans in the mempool.
21. **`rejectedTxns` scope.** Non-rule (internal) errors also mark a tx rejected until the next network block (`internal/netsync/manager.go:1009`). Local `ProcessBlock` (`:2197-2215`) does not reset the filter.
22. **`PruneStakeTx` returns silently on an agenda-query error** (`internal/mempool/mempool.go:2281-2285`). That also skips `PruneOldVotes`.
23. **No main mempool size cap** (see I11).

**Network, RPC and mixing**

24. **getdata pending counter for unknown inv types.** In `handleServeGetData` the `default:` branch (`server.go:601-604`) does `continue` without decrementing `numPendingGetDataItemReqs` (decrements at `:621, 649`). Wire does not restrict `InvVect.Type` (`wire/invvect.go:74-75`). Inferred effect: per-peer drift toward the 100000 disconnect limit.
25. **`sendrawmixmessage` relays even when the mixpool returned nothing** (orphan or duplicate) (`internal/rpcserver/rpcserver.go:4384-4393`; `rpcadaptors.go:441-443`). Orphans are not servable (`mixing/mixpool/mixpool.go:341-352`).
26. **Reconsidered mixpool orphans skip some checks.** They bypass the same-type/same-session conflict check (`mixing/mixpool/mixpool.go:1310-1321` vs `1561-1657`). Reconsidered KEs skip the `checkAcceptKE` time window.
27. **`clientcert` auth with an empty `--clientcafile`** (`config.go:718-722, 1046-1055`; `server.go:3689`; `internal/rpcserver/rpcserver.go:5521-5523`). Could TLS then not require client certs, while `checkAuth` admits everyone?
28. **Config bounds.** `RPCMaxConcurrentReqs = 0` is accepted (`config.go:1070`), giving an unbuffered semaphore (`internal/rpcserver/rpcwebsocket.go:62-63, 1549`). A negative `--maxpeers` is not validated (`server.go:3931`).
29. **Check-then-act on RPC client limits** (`internal/rpcserver/rpcserver.go:5942-5947`; `internal/rpcserver/rpcwebsocket.go:113-128`). Whitelisted inbound peers bypass `MaxNormalConns` (`internal/connmgr/connmanager.go:1693`).
30. **`MisbehavingBlock` voting suppression** (`server.go:2954-2957`) depends on node-local strike data (`mixing/mixpool/observer.go:18`). Is this intended policy?
31. **Root `go.mod` replace for a non-existent `./limits`** (`go.mod:85`). It has no build effect; is it a stale entry?
32. **`OnRead` bans on `ErrUnknownCmd`** (`server.go:1853-1856`). Forward-compatibility implications?

---

## Suggested Follow-Up Review Areas

The list is prioritised by architectural complexity, number of dependents, state
responsibility, and how thin the existing tests are.

1. **Block acceptance pipeline: `ProcessBlock` ↔ header-first ↔ data-commitment attribution** (`internal/blockchain/process.go:445-665`, `internal/blockchain/validate.go:881-2143`)
   - *Why:* recently restructured (024899e, ddd3b6d, b56e78f, ead8ba7). It decides permanent invalid-marking, which affects every peer and RPC path, and it adds a new cache.
   - *Verify:* use the chaingen harness to cover:
     - mutated-data vs bad-header blocks;
     - assume-valid and fast-add negative cases;
     - invalidate/reconsider with `recentMerkleChecks`;
     - out-of-order data delivery;
     - differential runs of `TestFullBlocks` before and after these commits.
2. **Agenda resolution consistency: validation ↔ mining ↔ mempool ↔ RPC ↔ DB upgrade** (`agendas.go`, `thresholdstate.go`, `server.go:3550, 4109`, `internal/blockchain/upgrade.go:2648`)
   - *Why:* a large refactor with hard-coded historical anchors, and five separate consumers.
   - *Verify:*
     - Replay mainnet and testnet3 headers and compare positional, anchor and tally results for every agenda.
     - Build side-chain fixtures below and above the anchors.
     - Assert that RPC `getvoteinfo` agrees with `isAgendaActive`.
3. **Reorg and recovery: chain ↔ UtxoCache ↔ ffldb ↔ stake undo** (`internal/blockchain/chain.go:586-1397`, `internal/blockchain/utxocache.go:681-1063`, `ffldb/reconcile.go`, `blockchain/stake/tickets.go:740-1040`)
   - *Why:* highest state-management responsibility, spread across three stores (flat files, leveldb metadata cache, UTXO leveldb). Crash consistency is not exercised by unit tests.
   - *Verify:*
     - Fault-injection or kill-point tests during connect, disconnect and flush.
     - Deep-reorg tests that span `minMemoryStakeNodes`.
     - Compare the UTXO set hash after recovery with the hash from a clean replay.
4. **Mempool vote admission and the template generator** (`internal/mempool/mempool.go:259-398, 1072-1203`, `internal/mining/bgblktmplgenerator.go:790-1103`, `internal/mining/mining.go:1251-1333`)
   - *Why:* new vote tracker, new lock dependencies (mempool → tracker → `chainLock`), pruning thresholds, and an `OnVoteReceived` ordering question.
   - *Verify:*
     - `go test -race` across the mempool, mining and netsync packages.
     - Simnet scenarios with competing tips, late votes and forced head reorganizations.
     - Check vote counts against the tracker after each event.
5. **Peer message handling ↔ resource bounds** (`server.go:528-1906`, `peer/peer.go:1086-1857`, `wire/message.go`)
   - *Why:* this is the externally reachable surface. There are no native fuzz tests, and Open Questions 24 and 32 concern this area.
   - *Verify:*
     - Add `go test -fuzz` harnesses for `ReadMessageN` and the `MsgTx`/`MsgBlock`/mix decoders.
     - Run adversarial-peer harness tests for getdata, inv, notfound and headers floods.
     - Check that counters return to zero.
6. **Mixpool acceptance and its coupling to the mempool and voting** (`mixing/mixpool/mixpool.go:1155-1724`, `observer.go`, `server.go:1754, 2954, 3752, 4132`)
   - *Why:* many new rules came in with few new tests. The orphan reconsideration path differs from direct acceptance. It affects mempool admission and vote notifications.
   - *Verify:*
     - Table tests for the identity cap, late, duplicate and whole-minute KEs, and the >64-peer KE.
     - Orphan-reconsideration parity tests.
     - An end-to-end simnet mix run with a misbehaving participant.
7. **netsync sync-height and request gating** (`internal/netsync/manager.go:393-409, 655-837, 1216-1370, 1506-1786`)
   - *Why:* 17dd9f4 has no test. `SyncHeight` feeds `IsCurrent`, which gates tx relay, mining and RPC. There is no penalty for invalid blocks.
   - *Verify:*
     - Unit tests with mock peers that claim inflated heights before and after header sync.
     - Peer disconnect and reassignment of in-flight blocks.
     - `IsCurrent` transitions.
8. **RPC authentication and limits configuration matrix** (`config.go:1021-1220`, `internal/rpcserver/rpcserver.go:5434-5947`, `internal/rpcserver/rpcwebsocket.go:1476-1562`)
   - *Why:* this is a privileged control surface. The limited allowlist includes state-affecting calls, and there are configuration edge cases (Open Questions 27–29).
   - *Verify:*
     - Run the `rpctest` integration suite, which is not in CI.
     - Add table tests over auth-mode × TLS × credentials × limit values.
     - Load-test concurrent connections at the client and websocket limits.
