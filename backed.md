# Backed on MegaETH

## What is Backed?

**Backed** is a MegaETH product stack for **fundraising attached to on-chain agent identities** (ERC-8004). A creator who owns an `agentId` can spin up a **raise**: the system deploys, in one step, a **Gnosis Safe** treasury, a **Sale** contract for a time-bounded, capped collateral round, and an **AgentExecutor** module on the Safe so a fixed **operator** can execute treasury actions only through an **allowlist + per-function policy**—not arbitrary transfers.

**For participants:** during the sale window, investors **commit** an accepted ERC-20 (e.g. USDM). After the window ends, anyone can **finalize**. If commitments reach the minimum, the raise **succeeds**: collateral goes to the Safe, a fixed-supply **vault share token** (`AgentVaultToken`) is bootstrapped, and investors **claim** shares (not the collateral back). If the raise **fails**, investors **refund**. Later phases follow vault rules (lockup, settlement, redemption) as implemented on chain.

**What Backed is not:** it is not a general-purpose DAO framework or a permissionless treasury product. It is a **narrow pipeline**: identity-gated creation → admin-approved sale → deterministic finalize → module-gated post-raise execution.

---

This document is for AI-assisted **integration and debugging** of the Backed contracts on MegaETH.

## When to Use This Document

- **`AgentRaiseFactory`** — create a raise from an **ERC-8004** `agentId`; deploys Safe + **`Sale`** + **`AgentExecutor`**
- **`Sale`** — `commit` / `finalize` / `claim` / `refund`
- **`AgentVaultToken`** — ERC-4626-style vault shares with **fixed supply** minted once after a successful raise
- **`AgentExecutor`** + **`ContractAllowlist`** — allowed targets and selector policy for calls routed through the **module** path
- **`backed` CLI** — resolves deployment addresses from manifest JSON

## Verified Deployment Constants (official manifests)

Confirm on [MegaETH testnet explorer](https://megaeth-testnet-v2.blockscout.com) / [MegaETH mainnet explorer](https://mega.etherscan.io) before production; addresses change when redeployed.

| Item | MegaETH testnet (6343) | MegaETH mainnet (4326) |
|------|-------------------------|-------------------------|
| `SafeModuleSetup` | `0x7b6EbB0ede8ac0224a176663e6c07Dece0a37010` | `0x54459A9431bD98c754180DEB32B067Cf31bDfF33` |
| `ContractAllowlist` | `0x54459A9431bD98c754180DEB32B067Cf31bDfF33` | `0x577be362178d20A3370722807d0294fA5D8A5a2A` |
| `AgentRaiseFactory` | `0x577be362178d20A3370722807d0294fA5D8A5a2A` | `0x45179eE92887e5770E42CD239644bc7b662673af` |
| `USDM` (common collateral reference) | `0x9f5A17BD53310D012544966b8e3cF7863fc8F05f` | `0xFAfDdbb3FC7688494971a79cc65DCa3EF82079E7` |

The **ERC-8004 identity registry** address is network-specific; take it from current MegaETH docs or your deployment manifest. The factory requires `IDENTITY_REGISTRY.ownerOf(agentId) == msg.sender` when calling `createAgentRaise`.

## End-to-End Flow

1. **Create** — `createAgentRaise(...)` deploys **Safe**, **`Sale`**, **`AgentExecutor`** and wires the module.
2. **Approve** — platform admin must **approve** the project or `commit` reverts.
3. **Fundraise** — investors **`commit`** within the window; totals are capped at **`MAX_RAISE`**.
4. **Finalize** — after `endTime`, anyone calls **`finalize()`**; outcome depends on totals vs **`MIN_RAISE`**.
5. **Success** — bootstrap **`AgentVaultToken`**, move accepted funds to the Safe, mint fixed shares to the sale for **`claim()`**.
6. **Failure** — investors use **`refund()`**.
7. **After the raise** — the operator uses **`AgentExecutor.execute`** under allowlist and selector rules (module path only).

Trust and admin assumptions are explicit in the project security model (identity registry, admin admission, Safe setup, operator behavior).

## MegaETH Integration Notes

- Prefer **`eth_sendRawTransactionSync`** (EIP-7966) for writes to avoid receipt polling.
- Use **remote `eth_estimateGas`**; MegaEVM differs from vanilla EVM.
- Heavy read UIs: batch **`eth_call`** via Multicall (see main skill).

## Core Flows (contracts)

### Create raise

`AgentRaiseFactory.createAgentRaise(...)`: validates `ownerOf(agentId)`, collateral, schedule, scaled min/max; deploys Safe, Sale, Executor; stores metadata.

### Fundraising

- **`commit`** needs **approval**, open window, and correct collateral handling.
- **Oversubscription** is handled at **`claim`**, not at `commit`.

### Finalize

- **`finalize()`** is permissionless after `endTime`.
- Below **`MIN_RAISE`** ⇒ failed; at or above ⇒ success path with vault bootstrap.

### Investors

- Success: **`claim()`** → **shares**, not underlying collateral.
- Failure: **`refund()`**.
- Emergency paths (e.g. `emergencyRefund`) depend on deployment ABI.

### Lockup and settlement

**`LOCKUP_END_TIME`** ties lockup to sale end + configured duration; redemption follows **`AgentVaultToken`** and treasury settlement rules.

### Treasury execution

- Only the immutable **`AGENT`** may call **`AgentExecutor.execute`**.
- With **`allowlistEnforced`**, targets and selectors must be allowed.
- **`allowlistEnforced == false`** relaxes checks except hard-blocked addresses.
- Safe **owners** may still have powers outside the module policy.

## CLI and Config

The **`backed`** CLI reads addresses from `deployment.*.json` (and `backend/deployments/` mirrors). Run **`backed network`** to verify.

## Common Mistakes

- Treating an unapproved project as live for **`commit`**.
- Expecting **`claim`** to return collateral on success — it allocates **shares**.
- Equating **executor policy** with full Safe control.
- Using **stale addresses** from old deployment JSON.
- Ignoring **collateral decimals** when interpreting raise caps in the UI.

## Source Material

- Add canonical public repo and user-facing docs URLs here when published.
