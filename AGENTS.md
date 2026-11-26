# AGENTS.md

## Repo shape
- Foundry-only Solidity repo; no package manager, task runner, CI, or lint/typecheck config. Minimal `foundry.toml`; libs are git submodules pinned in `foundry.lock` (`lib/forge-std`, `lib/openzeppelin-contracts`). Clone with `--recurse-submodules`; if `lib/` is empty, run `forge install`.
- Current vault is `src/CovenantVaultV31.sol`. `src/CovenantVaultV3.sol` and `src/CovenantVault.sol` (v2) are retained standalone versions, not base contracts; `legacy/CovenantVaultV1.sol` is v1.
- `ICovToken`, `IPriceSource`, `IBuybackRouter` are declared in `src/CovenantVaultV3.sol`; v3.1 and `src/mocks/MockBuyback.sol` import them. Changing those interfaces affects all consumers.

## Commands
- Verify with `forge test` (74 tests, including fuzz). Focus: `forge test --match-path test/CovenantVaultV31.t.sol`, `forge test --match-contract CovenantVaultV31Test`, `forge test --match-test test_buyback_belowThreshold_burnsOnly20Percent`.
- `forge fmt --check` has pre-existing diffs at current HEAD in `src/` and `test/`; do not bulk-format unrelated files.
- Deploy current version: `source .env && forge script script/DeployV31.s.sol --rpc-url base_sepolia --broadcast --account <your-keystore>`. RPC aliases in `foundry.toml`: `base` is mainnet, `base_sepolia` is testnet; never use `base` for test deploys.
- Deploy env: required `USDC_ADDRESS`, `TREASURY`, `STABILIZATION_VAULT`; optional `COV_INITIAL_SUPPLY`, `COV_PER_USDC`, `ROUTER_RESERVE_COV`. Never put a private key in `.env`; use `cast wallet new ~/.foundry/keystores <name>`.

## Invariants and gotchas
- `setCovUtilityToken` is one-shot and records `initialCovSupply`; call it immediately after token deploy, before any burns. v3.1 decay starts only when supply is strictly below 50% of that recorded supply.
- USDC fees split 60% treasury / 40% `pendingBuybackUsdc`; `executeBuyback` may spend only that queue, never household balances. USDC is physically pooled but tracked per household in `households[id].stablecoinBalance`.
- 90-day dissolution: only the non-initiator can cancel; the caller of the final split pays the fixed divorce fee, and the 50/50 split gives odd units to partner B. Child trusts are timestamp records, not funded sub-vaults; no birth verification exists.
- v3.1 decay vs v3: below threshold, COV-paid fees and buyback burn 20% (rest to stabilization); voluntary boosts always burn 100%. v3 used a fixed 5M threshold and a different `BuybackExecuted` event signature.
- Tests use 6-decimal `TestUSDC`, real `CovToken`, `MockPriceSource`, and a pre-funded `MockBuybackRouter`; `router.setDeliveryBps` simulates bad fills. Preserve that setup when adding tests.
- Testnet only, unaudited: price source and router are owner-set mocks and the owner is a single key. Mainnet needs an oracle/TWAP, real DEX adapter, multisig + timelock, and audit.
- Ticker is COVP in code/docs; the already-deployed testnet token still shows COV on explorers. Do not "fix" deployment addresses or notes to match.
