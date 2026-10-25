# Reya (proposed connector)

!!! warning "Proposed — not available in a Hummingbot release"
    This page describes the draft `reya_perpetual` connector in [Hummingbot PR #8509](https://github.com/hummingbot/hummingbot/pull/8509). It is not an announcement of supported trading. Deployment of the proposed risk API, devnet credential and lifecycle QA, and connector admission and maintenance arrangements remain pending.

## 🛠 Connector Info

- **Exchange Type**: Decentralized Exchange (**DEX**)
- **Market Type**: Central Limit Order Book (**CLOB**)

| Component | Status | Notes |
| --------- | ------ | ----- |
| Spot Connector | Not included | |
| [Perp Connector](#perp-connector) | Proposed | Draft implementation; release gates below |
| Spot Candles Feed | Not included | |
| Perp Candles Feed | Not included | |

## ℹ️ Exchange Info

- **Website**: <https://reya.xyz/>
- **API Docs**: <https://docs.reya.xyz/developers>
- **Public API Specification**: <https://github.com/Reya-Labs/reya-api-specs>
- **Proposed Risk API**: [reya-api-specs PR #119](https://github.com/Reya-Labs/reya-api-specs/pull/119)

## 🔑 Connection Configuration (draft)

The proposed connector uses an owner wallet address, a perpetual account ID, and a dedicated signer already authorized for that account. It does not create an account or provision signer authorization. Reserve the account for one bot and do not share its signer with another process: signed requests require increasing nonces.

The draft `connect reya_perpetual` flow uses these fields:

| Field | Value |
| ----- | ----- |
| `reya_perpetual_private_key` | Private key of the dedicated, authorized signer |
| `reya_perpetual_wallet_address` | Owner wallet address, which may differ from the signer address |
| `reya_perpetual_account_id` | Perpetual account ID belonging to that owner |

!!! warning "Credential QA remains pending"
    The connector verifies account ownership through the account summary and derives the signer address from the configured private key. It accepts the owner's signer or checks a delegate against `enabledApiWallets` from Reya's read-only `GET /api/trading/wallet/{address}/apiWalletStatus` endpoint, comparing addresses case-insensitively. Missing keys, malformed permission responses and failed permission reads reject validation. This check submits no order and makes no direct RPC call. Valid and invalid credential flows still require devnet QA.

Restore Hummingbot's saved order tracking state after a restart. As a conservative local policy, the draft connector withholds new order budget when it finds untracked open orders. It does not automatically resubmit an order whose outcome is uncertain.

## 🔀 Perp Connector

- **ID**: `reya_perpetual`
- **Connection Type**: REST / WebSocket
- **Trading Pair Format**: `BASE-RUSD`, for example `BTC-RUSD`
- **Implementation**: [Draft connector PR #8509](https://github.com/hummingbot/hummingbot/pull/8509)
- **Network**: The current draft implements mainnet only. It has no selectable testnet or devnet domain.

### Order Types

The draft implements these `OrderType` constants:

| Order Type | Venue Behavior |
| ---------- | -------------- |
| `LIMIT` | Good-till-cancelled limit order |
| `LIMIT_MAKER` | Maker-only limit order |
| `MARKET` | Immediate-or-cancel limit order with a 0.5% price bound; execution may be partial or absent |

Reya permits reduce-only perpetual orders only with immediate-or-cancel execution. Closing a position with `PositionAction.CLOSE` requires `MARKET`; limit closes are rejected locally. Stop orders are not implemented.

### Position Modes and Leverage

Only one-way positions with cross margin are implemented. Hedge mode and isolated margin are not supported. Leverage is a local budgeting preference bounded by the market's initial margin requirement; the connector does not change an exchange-side leverage setting.

### Balances and Available Margin

The connector depends on the new `GET /v2/wallet/{address}/accountSummaries` endpoint and additional risk fields on the existing `GET /v2/wallet/{address}/positions` endpoint proposed in [API spec PR #119](https://github.com/Reya-Labs/reya-api-specs/pull/119). Both changes must be deployed before the connector can use the margin and position valuations for trading.

Balances represent account-level margin value in RUSD after collateral haircuts. Reya currently does not reserve or consume margin for resting orders; the API's `freeCollateral` reflects that venue model.

The connector clamps free collateral at zero and subtracts its own conservative reservations for tracked orders until settlement reconciliation completes. Those reservations and the budget freeze for untracked open orders are local connector policies. They do not represent exchange-held margin or a missing API field. Resting orders are distinct from matched executions awaiting settlement, which the connector reconciles as described below.

Missing or stale required position risk data independently withholds order budget, even when a balance refresh succeeds. Stale prices or failed margin reads also withhold budget. This budget is not a withdrawal amount or a guaranteed order-size quote. Transfers and withdrawals are not implemented.

The proposed position stream can publish valuation updates without a new execution. Price-only updates leave a still-fresh budget usable until REST reconciliation; position changes, unavailable risk, cash or execution events, and stream failures withhold budget immediately. Reconciliation runs at most once every five seconds, including periodic checks while the stream is healthy. Responses started before an invalidating event cannot restore freshness. These stream changes remain part of the undeployed API proposal, and valuation messages do not create trade fills.

### Settlement, Fees and Funding

Matching is provisional on Reya. The connector reports fills from settled executions rather than matched order quantities, and reconciles execution busts. Orders remain tracked until their terminal settlement state is reconciled.

Fee budgets use the account's fee tier without discounts; settled executions supply actual fees. Funding accrues continuously in realized balances. The connector does not emit synthetic periodic funding-payment events.

### Release Gates

Before this connector can be listed as available:

- Deploy the account summaries endpoint and enriched positions response, and validate their integration.
- Verify owner, delegate and invalid credential cases on devnet, including permission-read failures.
- Enable a devnet validation path and complete signed create, cancel, partial-fill, execution-bust, reconnect and restart QA with a dedicated account. The current draft has no devnet selector, and signed lifecycle validation remains pending.
- Complete Hummingbot's [perpetual connector developer checklist](https://hummingbot.org/connectors/connectors/perp-connector-checklist/), [QA checklist](https://hummingbot.org/connectors/connectors/test-perp/) and [connector admission and maintenance process](https://hummingbot.org/connectors/connectors/).

Offline tests do not establish production readiness. No live orders have been submitted as part of the connector's automated validation.
