# File-by-File Reading Guide

A reading plan organized by **execution order** -- from process startup through runtime steady-state -- so you encounter each file in roughly the same sequence the code itself runs.

---

## Phase 1: Entry Point & CLI Bootstrap

| # | File | What to look for |
|---|------|-----------------|
| 1 | `cmd/inferenced/main.go` | `main()` creates the root Cobra command via `cmd.NewRootCmd()` and calls `Execute()`. This is the only `func main` in the project. |
| 2 | `cmd/inferenced/cmd/root.go` | `NewRootCmd()` wires dependency-injection (`depinject`), configures the SDK (`initSDKConfig`), sets up the keyring (`ProvideKeyring`), and returns the root command. |
| 3 | `cmd/inferenced/cmd/commands.go` | `initRootCmd()` registers every sub-command: genesis helpers, query, tx, keys, CometBFT, etc. `newApp()` is the factory that ultimately calls `app.New()`. `appExport()` handles state export. |
| 4 | `cmd/inferenced/cmd/config.go` | Configuration file loading and defaults. |
| 5 | `cmd/inferenced/cmd/toml_handling.go` | Low-level TOML parsing used by config helpers. |

### CLI commands you'll encounter but can skim on first read

| File | Purpose |
|------|---------|
| `cmd/inferenced/cmd/custom_gentx.go` | Custom genesis-transaction generation |
| `cmd/inferenced/cmd/patch_genesis.go` | Patches genesis JSON after initial creation |
| `cmd/inferenced/cmd/download_genesis.go` | Downloads genesis from a running network |
| `cmd/inferenced/cmd/register_participant_command.go` | Registers an ML participant from CLI |
| `cmd/inferenced/cmd/create_client_command.go` | Creates a client context for RPC |
| `cmd/inferenced/cmd/sign_command.go` | Offline signature operations |
| `cmd/inferenced/cmd/set_seeds.go` | Writes seed-node addresses to config |
| `cmd/inferenced/cmd/sync_snapshots.go` | State-sync snapshot setup |

---

## Phase 2: Application Construction

When `newApp()` (from Phase 1) is called at node start, it enters the app package.

| # | File | What to look for |
|---|------|-----------------|
| 6 | `app/app_config.go` | `AppConfig()` defines the full depinject module graph: which Cosmos SDK and custom modules are included, store keys, and inter-module accounts. Read this to understand module ordering. |
| 7 | `app/app.go` | The `App` struct holds every keeper. `New()` (~line 219) is the constructor: it creates stores, instantiates keepers in dependency order, registers hooks between modules, sets the ante handler, and wires the ABCI interface. |
| 8 | `app/legacy.go` | Registers IBC-related modules that don't use depinject yet (Capability, IBC core, Transfer, ICA). |
| 9 | `app/ante.go` | `HandlerOptions` and the ante-handler chain. Key decorator: `LiquidityPoolFeeBypassDecorator` lets certain WASM contract calls skip gas fees. Follow `isAllWasmExec()` and `matchesAllowedSwap()` for the bypass logic. |

---

## Phase 3: Genesis Initialization

On first start (or `init` / `genesis` commands), genesis state is loaded.

| # | File | What to look for |
|---|------|-----------------|
| 10 | `app/genesis.go` | `GenesisState` type definition -- a `map[string]json.RawMessage` keyed by module name. |
| 11 | `app/genesis_account.go` | Handles genesis-time account creation with vesting schedules. |
| 12 | `x/inference/module/genesis.go` | `InitGenesis()` creates epoch 0, initializes holding accounts, sets tokenomics data, imports participants/models/bridge addresses from the genesis file, and configures guardians. |
| 13 | `x/inference/types/genesis.go` | Genesis state structure for the inference module. |
| 14 | `x/bls/module/genesis.go` | BLS module genesis -- clears active epoch ID. |
| 15 | `x/collateral/module/genesis.go` | Collateral module genesis -- initializes epoch tracker. |
| 16 | `x/restrictions/module/genesis.go` | Restrictions module genesis -- sets transfer-restriction parameters. |
| 17 | `x/streamvesting/module/genesis.go` | Stream-vesting genesis -- loads vesting schedules. |
| 18 | `x/genesistransfer/module/genesis.go` | Genesis-transfer module genesis -- initial token distribution records. |
| 19 | `x/bookkeeper/module/genesis.go` | Bookkeeper genesis -- logging configuration. |
| 20 | `x/inference/module/genesis_guardian_enhancement.go` | Guardian system setup at genesis time. |

---

## Phase 4: Module Lifecycle (BeginBlock / EndBlock)

Once the chain is running, CometBFT calls `BeginBlock` and `EndBlock` every block.

| # | File | What to look for |
|---|------|-----------------|
| 21 | `x/inference/module/module.go` | `AppModule` implements `BeginBlock` and `EndBlock`. **BeginBlock**: calls `UpdateDynamicPricing()`. **EndBlock**: handles confirmation PoC, epoch-phase transitions, inference expiration, reward settlement, pruning, and partial/full upgrades. This is the heartbeat of the chain. |
| 22 | `x/inference/keeper/dynamic_pricing.go` | `UpdateDynamicPricing()` -- recalculates per-model inference prices at block start. |
| 23 | `x/inference/module/confirmation_poc.go` | `handleConfirmationPoC()` -- processes proof-of-compute confirmation events in EndBlock. |
| 24 | `x/inference/module/model_assignment.go` | Model-to-participant assignment logic run during epoch transitions. |
| 25 | `x/inference/module/top_miners.go` | Top-miner tracking updated at epoch boundaries. |
| 26 | `x/inference/module/power_capping.go` | Caps individual participant power to prevent centralization. |
| 27 | `x/inference/module/chain_validation.go` | Chain-level validation checks during EndBlock. |
| 28 | `x/inference/module/staking_hooks.go` | Hooks into the staking module for validator-set changes. |
| 29 | `x/inference/keeper/pruning.go` | `Prune()` -- removes old inferences, epochs, and PoC data. |
| 30 | `x/inference/keeper/partial_upgrade.go` | Tracks rolling node-version upgrades without a full chain halt. |

---

## Phase 5: Epoch Lifecycle

Epochs are the macro unit of time. Understand the phase system before diving into transactions.

| # | File | What to look for |
|---|------|-----------------|
| 31 | `x/inference/types/epoch_context.go` | `EpochContext` -- computes phase boundaries: `StartOfPoC()`, `EndOfPoCGeneration()`, `StartOfPoCValidation()`, `PoCExchangeWindow()`, `ValidationExchangeWindow()`. This is the single source of truth for epoch timing. |
| 32 | `x/inference/types/epoch_stages.go` | `EpochStages` -- computes all block heights for epoch phases, used by APIs and monitoring. |
| 33 | `x/inference/types/epoch_params.go` | Epoch duration, phase lengths, and other epoch-level parameters. |
| 34 | `x/inference/types/current_epoch_stats.go` | Per-epoch aggregate statistics. |
| 35 | `x/inference/keeper/epoch.go` | CRUD operations for epoch state in the KV store. |
| 36 | `x/inference/keeper/epoch_group_data.go` | Per-model participant groups within an epoch. |
| 37 | `x/inference/keeper/epoch_group_validations.go` | Validation results stored per epoch group. |
| 38 | `x/inference/epochgroup/epoch_group.go` | `EpochGroup` and `EpochMember` -- how participants are grouped, weighted, and assigned to models. `NewEpochMemberFromActiveParticipant()` and `NewEpochMemberFromStakingValidator()` show how members enter a group. |
| 39 | `x/inference/epochgroup/voting.go` | Voting/consensus logic within an epoch group. |
| 40 | `x/inference/epochgroup/random.go` | Randomization used for group assignment. |
| 41 | `x/inference/epochgroup/unit_of_compute_price.go` | Per-group pricing calculations. |

---

## Phase 6: Inference Lifecycle (Core Transaction Flow)

The main business flow: a client requests inference, a node executes it, validators verify it.

### 6a. State & types

| # | File | What to look for |
|---|------|-----------------|
| 42 | `x/inference/keeper/keeper.go` | The `Keeper` struct and all its `collections.*` fields. This is the schema of the entire inference module's on-chain state. |
| 43 | `x/inference/types/types.go` | Core domain types: `Participant`, `Inference`, `Model`, `Epoch`, `EpochGroupData`, `EpochGroupValidations`, `PoCBatch`, `RandomSeed`, `TopMiner`, `SettleAmount`. |
| 44 | `x/inference/types/keys.go` | Storage key prefixes 0-35. Tells you where each data type lives in the KV store. |
| 45 | `x/inference/types/inference.go` | Methods on the `Inference` type -- status helpers, completion checks. |
| 46 | `x/inference/types/errors.go` | All sentinel errors for the module. |
| 47 | `x/inference/types/expected_keepers.go` | Interfaces the inference module requires from other modules (Bank, Staking, BLS, Collateral, etc.). |

### 6b. Transaction handlers (read in lifecycle order)

| # | File | What to look for |
|---|------|-----------------|
| 48 | `x/inference/keeper/msg_server.go` | `msgServer` struct that implements the gRPC `MsgServer` interface. |
| 49 | `x/inference/keeper/msg_server_start_inference.go` | `StartInference` -- validates participants and signatures, records model price, calls `calculations.ProcessStartInference()`, locks escrow, sets timeout. |
| 50 | `x/inference/keeper/msg_server_finish_inference.go` | `FinishInference` -- validates result hash, pays executor, marks inference complete. |
| 51 | `x/inference/keeper/msg_server_invalidate_inference.go` | `InvalidateInference` -- dispute path when a result is contested. |
| 52 | `x/inference/keeper/msg_server_revalidate_inference.go` | `RevalidateInference` -- re-runs validation after dispute. |
| 53 | `x/inference/keeper/msg_server_validation.go` | `MsgValidation` -- validators submit validation scores for completed inferences. |

### 6c. Supporting state operations

| # | File | What to look for |
|---|------|-----------------|
| 54 | `x/inference/keeper/inference.go` | `SetInference`, `GetInference`, iteration helpers. |
| 55 | `x/inference/keeper/inference_timeout.go` | Timeout tracking -- `addTimeout`, expiration checks in EndBlock. |
| 56 | `x/inference/keeper/inference_validation_details.go` | Detailed per-inference validation records. |
| 57 | `x/inference/keeper/payment_handler.go` | Escrow lock/release, fee distribution. |
| 58 | `x/inference/keeper/activeparticipants.go` | Queries and filters for currently active participants. |

---

## Phase 7: Proof of Compute (PoC)

PoC ensures nodes actually performed the computation they claim.

| # | File | What to look for |
|---|------|-----------------|
| 59 | `x/inference/keeper/msg_server_submit_poc_batch.go` | Submits a batch of PoC evidence during the generation phase. |
| 60 | `x/inference/keeper/msg_server_submit_poc_validation.go` | Other participants validate submitted PoC batches. |
| 61 | `x/inference/keeper/msg_server_submit_seed.go` | Submits the random seed for the epoch (ties into DKG). |
| 62 | `x/inference/keeper/poc_batch.go` | CRUD for PoC batch state. |
| 63 | `x/inference/keeper/random_seed.go` | Random seed storage. |
| 64 | `x/inference/keeper/confirmation_poc_event.go` | Confirmation PoC events for honest-node verification. |
| 65 | `x/inference/types/confirmation_poc_event.go` | Type definitions for confirmation PoC. |

---

## Phase 8: Calculations & Business Logic

Pure functions that compute scores, reputation, and settlement.

| # | File | What to look for |
|---|------|-----------------|
| 66 | `x/inference/calculations/inference_state.go` | Inference lifecycle state machine -- transitions and valid states. |
| 67 | `x/inference/calculations/reputation.go` | Reputation score formula. |
| 68 | `x/inference/calculations/sprt.go` | Sequential Probability Ratio Test -- statistical validation of inference results. |
| 69 | `x/inference/calculations/should_validate.go` | Determines whether a node is eligible to validate a given inference. |
| 70 | `x/inference/calculations/signature_validate.go` | Cryptographic signature verification. |
| 71 | `x/inference/calculations/stats.go` | Aggregate node statistics computation. |
| 72 | `x/inference/calculations/status.go` | Node status derivation from stats and behavior. |
| 73 | `x/inference/calculations/share_work.go` | Work distribution algorithm across participants. |
| 74 | `x/inference/calculations/maximum_invalidations.go` | Penalty cap calculations. |
| 75 | `x/inference/calculations/min_validation_average.go` | Minimum quality threshold for validation acceptance. |

---

## Phase 9: Participant & Model Management

| # | File | What to look for |
|---|------|-----------------|
| 76 | `x/inference/keeper/msg_server_submit_new_participant.go` | Registers a new ML node with collateral. |
| 77 | `x/inference/keeper/msg_server_submit_new_unfunded_participant.go` | Registers a node without upfront collateral. |
| 78 | `x/inference/keeper/msg_server_submit_hardware_diff.go` | Updates a node's hardware specifications. |
| 79 | `x/inference/keeper/msg_server_register_model.go` | Registers a new ML model on-chain. |
| 80 | `x/inference/keeper/participant.go` | Participant CRUD and iteration. |
| 81 | `x/inference/keeper/model.go` | Model CRUD. |
| 82 | `x/inference/keeper/hardware_node.go` | Hardware spec tracking. |
| 83 | `x/inference/keeper/mlnode_version.go` | ML node software version tracking. |
| 84 | `x/inference/keeper/developer_stats_developer.go` | Developer-facing statistics. |
| 85 | `x/inference/keeper/developer_stats_transfer_agent.go` | Transfer-agent statistics. |

---

## Phase 10: Rewards & Settlement

| # | File | What to look for |
|---|------|-----------------|
| 86 | `x/inference/keeper/msg_server_claim_rewards.go` | `ClaimRewards` -- participants claim earned rewards. |
| 87 | `x/inference/keeper/bitcoin_rewards.go` | Bitcoin-denominated reward calculations and distribution. |
| 88 | `x/inference/keeper/settle_amount.go` | Settlement amount CRUD. |
| 89 | `x/inference/types/settle_amount.go` | `SettleAmount` type and helpers. |
| 90 | `x/inference/keeper/msg_server_submit_unit_of_compute_price_proposal.go` | Proposes changes to per-model inference pricing. |
| 91 | `x/inference/keeper/unit_of_compute.go` | Unit-of-compute price state. |

---

## Phase 11: BLS / Threshold Cryptography

The BLS module runs a Distributed Key Generation protocol each epoch to produce threshold signatures.

| # | File | What to look for |
|---|------|-----------------|
| 92 | `x/bls/module/module.go` | Module lifecycle and DKG phase management. |
| 93 | `x/bls/keeper/keeper.go` | `SetActiveEpochID` / `GetActiveEpochID` -- minimal keeper tracking the current DKG epoch. |
| 94 | `x/bls/keeper/dkg_initiation.go` | DKG protocol initiation logic. |
| 95 | `x/bls/keeper/msg_server_dealer.go` | Dealer phase -- distributes key shares. |
| 96 | `x/bls/keeper/msg_server_verifier.go` | Verifier phase -- validates received shares. |
| 97 | `x/bls/keeper/msg_server_threshold_signing.go` | Threshold signature generation. |
| 98 | `x/bls/keeper/msg_server_group_validation.go` | Group-key validation via aggregate signature. |
| 99 | `x/bls/keeper/phase_transitions.go` | DKG phase state machine: Initiation -> Dealing -> Verification -> Signing -> Completion. |
| 100 | `x/bls/keeper/bls_crypto.go` | Low-level BLS cryptographic operations. |
| 101 | `x/bls/types/types.go` | `ThresholdSignatureEvent`, `GroupKeyValidationEvent`, DKG phase enums. |

---

## Phase 12: Collateral & Slashing

| # | File | What to look for |
|---|------|-----------------|
| 102 | `x/collateral/module/module.go` | Module definition and lifecycle. |
| 103 | `x/collateral/module/hooks.go` | Slash/jail hooks that other modules call into. |
| 104 | `x/collateral/keeper/keeper.go` | Collections: `CollateralMap`, `Jailed`, `SlashedInEpoch`, `UnbondingIM` with `ByParticipant` index. |
| 105 | `x/collateral/keeper/msg_server_deposit_collateral.go` | Locks collateral for a participant. |
| 106 | `x/collateral/keeper/msg_server_withdraw_collateral.go` | Begins unbonding and eventual withdrawal. |
| 107 | `x/collateral/keeper/slashing_test.go` | `SlashCollateral()`, `UnbondCollateral()` -- penalty mechanics (read the test for behavior spec). |
| 108 | `x/inference/keeper/collateral.go` | Inference module's integration with collateral for slash/jail events. |
| 109 | `x/inference/types/slash_reasons.go` | Enumeration of all slashable offenses. |

---

## Phase 13: Transfer Restrictions & Vesting

| # | File | What to look for |
|---|------|-----------------|
| 110 | `x/restrictions/keeper/keeper.go` | `SendRestriction()` hook called on every bank send. `GetTransferRestrictionStatus()`, `GetTransferExemptions()`. |
| 111 | `x/restrictions/keeper/send_restriction.go` | The actual restriction logic -- blocks or allows transfers based on policy. |
| 112 | `x/restrictions/keeper/msg_execute_emergency_transfer.go` | Emergency transfer override for restricted accounts. |
| 113 | `x/streamvesting/keeper/keeper.go` | Vesting schedule management -- progressive token unlock over time. |
| 114 | `x/streamvesting/types/types.go` | `VestingSchedule` -- duration, release rate, cliff. |
| 115 | `x/genesistransfer/keeper/keeper.go` | Genesis-time token distribution. |
| 116 | `x/genesistransfer/keeper/transfer.go` | Core transfer execution logic. |
| 117 | `x/genesistransfer/keeper/validation.go` | Validates transfers against vesting and restriction rules. |
| 118 | `x/genesistransfer/keeper/transfer_records.go` | Audit trail for genesis transfers. |

---

## Phase 14: Cross-Chain Bridge

| # | File | What to look for |
|---|------|-----------------|
| 119 | `x/inference/keeper/bridge.go` | Top-level bridge logic and contract interactions. |
| 120 | `x/inference/keeper/bridge_native.go` | Native-side bridge operations. |
| 121 | `x/inference/keeper/bridge_transaction.go` | Bridge transaction state tracking. |
| 122 | `x/inference/keeper/bridge_utils.go` | Bridge helper functions. |
| 123 | `x/inference/keeper/bridge_wrapped_token.go` | CW20 wrapped-token management on the inference chain side. |
| 124 | `x/inference/keeper/msg_server_bridge_exchange.go` | Token swap across chains. |
| 125 | `x/inference/keeper/msg_server_request_bridge_mint.go` | Requests minting of wrapped tokens. |
| 126 | `x/inference/keeper/msg_server_request_bridge_withdrawal.go` | Requests withdrawal back to origin chain. |
| 127 | `x/inference/keeper/msg_server_register_bridge_addresses.go` | Registers cross-chain contract addresses. |
| 128 | `x/inference/keeper/msg_server_approve_bridge_token_for_trading.go` | Enables a wrapped token for DEX trading. |
| 129 | `x/inference/keeper/msg_server_register_token_metadata.go` | Token metadata registration. |
| 130 | `x/inference/keeper/msg_server_register_liquidity_pool.go` | DEX liquidity pool registration. |
| 131 | `x/inference/keeper/msg_server_migrate_all_wrapped_tokens.go` | Token migration on upgrade. |
| 132 | `x/inference/keeper/liquidity_pool.go` | Liquidity pool state management. |
| 133 | `x/inference/keeper/migrations_bridge.go` | Bridge state migrations between versions. |

---

## Phase 15: Distributed Training

| # | File | What to look for |
|---|------|-----------------|
| 134 | `x/inference/keeper/msg_server_create_training_task.go` | Creates a distributed training job. |
| 135 | `x/inference/keeper/msg_server_assign_training_task.go` | Assigns training to a specific node. |
| 136 | `x/inference/keeper/msg_server_claim_training_task_for_assignment.go` | Node claims a training job. |
| 137 | `x/inference/keeper/msg_server_join_training.go` | Node joins a multi-node training session. |
| 138 | `x/inference/keeper/msg_server_join_training_status.go` | Updates training participation status. |
| 139 | `x/inference/keeper/msg_server_training_heartbeat.go` | Heartbeat during long-running training. |
| 140 | `x/inference/keeper/msg_server_set_training_allow_list.go` | Manages training access control. |
| 141 | `x/inference/keeper/msg_server_add_user_to_training_allow_list.go` | Adds user to training allowlist. |
| 142 | `x/inference/keeper/msg_server_remove_user_from_training_allow_list.go` | Removes user from training allowlist. |
| 143 | `x/inference/keeper/msg_server_set_barrier.go` | Sets synchronization barrier for distributed training. |
| 144 | `x/inference/keeper/msg_server_submit_training_kv_record.go` | Training progress key-value tracking. |
| 145 | `x/inference/training/training_sync.go` | Training synchronization primitives. |

---

## Phase 16: Upgrades

| # | File | What to look for |
|---|------|-----------------|
| 146 | `app/upgrades.go` | `setupUpgradeHandlers()` -- registers handlers for v0.2.2 through v0.2.5. `registerMigrations()` -- state migration registration. |
| 147 | `app/upgrades_enabled.go` | Build-tag-dependent upgrade logic. |
| 148 | `app/upgrades/v0_2_2/` | v0.2.2 migration. |
| 149 | `app/upgrades/v0_2_3/` | v0.2.3 migration. |
| 150 | `app/upgrades/v0_2_4/` | v0.2.4 migration -- adds confirmation PoC weight calculations. |
| 151 | `app/upgrades/v0_2_5/` | v0.2.5 migration -- legacy bridge state migration + confirmation weight migration. |
| 152 | `x/inference/keeper/msg_server_create_partial_upgrade.go` | Partial (rolling) upgrade without chain halt. |
| 153 | `x/inference/keeper/migrations_confirmation_weight.go` | Confirmation-weight state migration logic. |

---

## Phase 17: Queries (Read Path)

All query handlers live in `x/inference/keeper/query_*.go`. Key ones:

| # | File | What to look for |
|---|------|-----------------|
| 154 | `x/inference/keeper/query.go` | Query handler stub / base. |
| 155 | `x/inference/keeper/query_participant.go` | Participant details. |
| 156 | `x/inference/keeper/query_inference.go` | Inference status and history. |
| 157 | `x/inference/keeper/query_epoch_group_data.go` | Epoch group assignments. |
| 158 | `x/inference/keeper/query_epoch_group_validations.go` | Validation results per group. |
| 159 | `x/inference/keeper/query_inference_validation_details.go` | Detailed validation info for a specific inference. |
| 160 | `x/inference/keeper/query_model.go` | Model information. |
| 161 | `x/inference/keeper/query_dynamic_pricing.go` | Current model pricing. |
| 162 | `x/inference/keeper/query_settle_amount.go` | Reward settlement data. |
| 163 | `x/inference/keeper/query_top_miner.go` | Top-performing miner. |
| 164 | `x/inference/keeper/query_tokenomics_data.go` | Economic state. |
| 165 | `x/inference/keeper/query_training_task.go` | Training job details. |
| 166 | `x/inference/keeper/query_participant_current_stats.go` | Real-time node stats. |
| 167 | `x/inference/keeper/query_excluded_participants.go` | Jailed/slashed nodes. |
| 168 | `x/inference/keeper/query_confirmation_poc_event.go` | PoC confirmation events. |

---

## Phase 18: Bookkeeper (Audit Trail)

| # | File | What to look for |
|---|------|-----------------|
| 169 | `x/bookkeeper/keeper/keeper.go` | `SendCoins()` wraps bank sends with double-entry logging. `logTransaction()` writes journal entries. Configurable via `LogConfig` (double-entry on/off, simple-entry on/off). |
| 170 | `x/bookkeeper/module/module.go` | Module lifecycle. |

---

## Phase 19: Permissions & Utilities

| # | File | What to look for |
|---|------|-----------------|
| 171 | `x/inference/permissions.go` | Permission system -- who can call which message types. |
| 172 | `x/inference/utils/utils.go` | General utility functions. |
| 173 | `x/inference/utils/key_validation.go` | Key format validation. |
| 174 | `x/inference/utils/signature_and_url_validation.go` | Signature and URL validation helpers. |
| 175 | `x/inference/keeper/keeper_utils.go` | Keeper-level utility functions. |
| 176 | `x/inference/keeper/slice_utils.go` | Slice manipulation helpers. |
| 177 | `x/inference/keeper/params.go` | Module parameter access. |
| 178 | `x/inference/types/params.go` | Parameter definitions and defaults. |
| 179 | `x/inference/types/coin.go` | Coin/denomination helpers. |
| 180 | `x/inference/types/logging.go` | Structured logging utilities. |

---

## Phase 20: Network & RPC Internals

| # | File | What to look for |
|---|------|-----------------|
| 181 | `internal/rpc/client.go` | `GetTrustedBlock()`, `GetBlockHash()`, `GetNodeId()` -- helpers for state-sync and node setup. |
| 182 | `internal/rpc/models.go` | RPC response type definitions. |
| 183 | `app/export.go` | `ExportAppStateAndValidators()` -- full state export for genesis migration. |
| 184 | `docs/docs.go` | OpenAPI / Swagger documentation embedding. |

---

## Phase 21: Proto / Codec Registration

Each module has a `types/codec.go` that registers its message types with the Cosmos SDK codec:

| File | Module |
|------|--------|
| `x/inference/types/codec.go` | Inference |
| `x/bls/types/codec.go` | BLS |
| `x/collateral/types/codec.go` | Collateral |
| `x/restrictions/types/codec.go` | Restrictions |
| `x/streamvesting/types/codec.go` | Streamvesting |
| `x/genesistransfer/types/codec.go` | Genesistransfer |
| `x/bookkeeper/types/codec.go` | Bookkeeper |

AutoCLI configuration (auto-generated CLI from proto):

| File | Module |
|------|--------|
| `x/inference/module/autocli.go` | Inference |
| `x/bls/module/autocli.go` | BLS |
| `x/collateral/module/autocli.go` | Collateral |
| `x/restrictions/module/autocli.go` | Restrictions |
| `x/streamvesting/module/autocli.go` | Streamvesting |
| `x/genesistransfer/module/autocli.go` | Genesistransfer |
| `x/bookkeeper/module/autocli.go` | Bookkeeper |

---

## Phase 22: Testing Infrastructure

| # | File | What to look for |
|---|------|-----------------|
| 185 | `testenv/testenv.go` | Full test environment setup -- creates an in-memory app instance. |
| 186 | `testutil/keeper/inference.go` | Test keeper factory for the inference module. |
| 187 | `testutil/keeper/bls.go` | Test keeper factory for BLS. |
| 188 | `testutil/keeper/collateral.go` | Test keeper factory for collateral. |
| 189 | `testutil/keeper/expected_keepers_mocks.go` | Mock interfaces for all keeper dependencies. |
| 190 | `testutil/network/network.go` | Multi-node test network setup. |
| 191 | `testutil/sample/sample.go` | Sample data generators for tests. |
| 192 | `testutil/constants.go` | Shared test constants. |

---

## Suggested reading paths

**Quick overview (15 files):**
1 -> 2 -> 3 -> 7 -> 9 -> 12 -> 21 -> 31 -> 42 -> 43 -> 49 -> 50 -> 66 -> 92 -> 104

**Inference flow deep-dive:**
49 -> 50 -> 51 -> 52 -> 53 -> 54 -> 55 -> 57 -> 66 -> 67 -> 68 -> 69

**Epoch & PoC deep-dive:**
31 -> 32 -> 35 -> 38 -> 59 -> 60 -> 61 -> 64 -> 21 (EndBlock section)

**Economic system:**
86 -> 87 -> 88 -> 90 -> 22 -> 104 -> 105 -> 106 -> 107 -> 109

**Bridge & cross-chain:**
119 -> 120 -> 121 -> 124 -> 125 -> 126 -> 127 -> 132
