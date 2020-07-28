# Bonded Assets (bAssets)

The following specification allows for a reference implementation of tradable, tokenized staking positions (**bAssets**), featuring maximized usability and interoperability.

bAsset implementations should be deployed on the same chain as its corresponding base Asset, with the Bridge specification ensuring a unified interface for transfers across different chains.

## Required Properties

**Speculo** defines a standardized format for developing bAssets to ensure interoperability between different bAsset implementations. The following is a list of properties that all bAsset implementations should include.

### 1. Fungibility

In order to maximize liquidity and simplicity, bAssets should be made fungible across all validators, regardless of its underlying validator and their properties.

- All bAssets share the same risk profile and maintain fungibility, which means that slashing risk is equally shared by all bAsset holders.
- Block rewards from staked tokens controlled by the bAsset system are distributed to bAssets on a pro-rata basis.
- On a bAsset `Send` transaction, the protocol must credit all prior accrued rewards to the sender.

### 2. 1:1 Conversion Peg

To improve user experience, bAssets should maintain a one-to-one peg with its underlying vanilla Asset, mimicking the price of their underlying Assets.

**Blockchains without slashing**: For blockchains that do not slash, bAssets and Assets should always form a one-to-one peg, without requiring additional insurance or peg maintenance mechanisms.

**Blockchains with slashing**: For blockchains that do slash, the one-to-one peg of bAssets may break in a slashing event. However, for these cases, a peg recovery mechanism is required. The peg recovery mechanism should cause the peg to gravitate back to 1:1, assuming constant protocol usage. An additional **insurance mechanism** that fills up slashed tokens in the case of a slashing event is recommended.

### 3. Ease of Redemption

For fluid arbitrage between secondary markets and the primary market (i.e. the bAsset Contract), implementations must support frictionless bAsset to vanilla Asset redemptions.

- Redemption of bAssets should be fully executed within a predetermined time period.
- Any holder should be able to redeem their bAssets without significant loss.



## Protocol-User Interface: Messages

Implementations should support five basic operations: Mint, Redeem, ClaimRewards, Send, and ReportSlashing.

- A user can **mint** a bAsset by sending Assets to the contract, which results in the delegation of the underlying token.
- A user can **burn** a bAsset by sending a transaction to undelegate underlying tokens and redeem the Assets.
- A user can **claim rewards** for bAssets by sending a transaction to claim rewards from staked underlying tokens.
- A user can **send** a bAsset to a different address and claim all prior staking rewards up to that point.
- A user can **report a slashing event** to the protocol.

Given all five operations are asynchronous, operations are designed in an init / finish architecture for staking related transactions: (Delegate / Mint), (Undelegate / Redeem), (ClaimRewards / WithdrawRewards). Staking transactions are first initiated by the protocol’s staking logic, and state changes occur when the transaction is verified to have successfully completed by the finish transactions.

### Mint

- `Delegate{validator, amount}` - Moves `amount` Assets from the `env.sender` account to the `contract` account. Moved Assets are delegated to the `validator` account from the `contract` account. The `validator` field is optional and can be left blank if an algorithm for automatic delegations exist.

- `Mint{msg_proof, amount}` - Verifies the validity of a delegation request having size `amount` using `msg_proof`. Creates a corresponding number of new bAssets and adds them to the `env.sender` account.

### Claim Rewards

- `ClaimRewards{}` - Claims staking rewards from validators to the `contract` account.

- `WithdrawRewards{msg_proof}` - Verifies whether staking rewards have been claimed using `msg_proof`. Moves a corresponding amount of claimed rewards (denominated in Assets) to the `env.sender` account.

### Burn

- `Undelegate{validator, amount}` - Burns `amount` bAssets and relays a transaction that undelegates a corresponding number of Assets from the `validator` account. The `validator` field is optional and can be left blank if an algorithm for automatic undelegations exist.

- `Redeem{msg_proof, amount}` - Verifies the validity of an undelegation request using `msg_proof`. Moves undelegated Assets to the `env.sender` account. This message triggers the `ClaimRewards{}` message, crediting all previously accrued staking rewards to the `env.sender` account.

### Send

- `Send{contract, amount, msg}` - Moves `amount` tokens from the `env.sender` account to the `recipient` account. The `msg` will be passed to the `recipient` contract, along with the `amount`. This message triggers the `ClaimRewards{}` message, crediting all previously accrued staking rewards to the `env.sender` account.

### Report Slashing Events

- `ReportSlashing{msg_proof, validator, amount, block_height}` - Verifies the validity of a reported slashing event using `msg_proof`. Deducts `amount` Assets from the Map recording current delegations. This message triggers the insurance / peg recovery mechanism, updating the bAsset to Asset conversion rate.


## Protocol-User Interface: Queries

The protocol must also allow certain queries to be made, allowing other applications to be aware of their internals.

- `TotalbAssetSupply` - The total number of bAssets issued by the bAsset system. This value should be equal to `TotalAssetsDelegated` on initial system setup (maintaining a one-to-one peg), but upon further usage, this should be equal to `TotalAssetsDelegated` * `ExchangeRatio`.

- `TotalAssetsDelegated` - The total number of assets delegated and being managed by the bAsset system. On a slashing event, this value will decrease, invoking either insurance or peg maintenance logic.

- `ExchangeRatio` - The bAsset to vanilla Asset exchange rate, updated in the case of staked vanilla Assets being slashed. This value is managed by the peg maintenance logic, and should be calculated as `TotalbAssetSupply` / `TotalAssetsDelegated`. 

- `Balance{Account}` - The number of bAssets being held by a particular user account.
