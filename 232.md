BLACK-PANDA-REACH

medium

# `BalancedVault` doesn't consider potential break in one of the markets

## Summary
In case of critical failure of any of the underlying markets, making it permanently impossible to close position and withdraw collateral all funds deposited to balanced Vault will be lost, including funds deposited to other markets.

## Vulnerability Detail

As Markets and Vaults on Perennial are intented to be created in a permissionless manner and integrate with external price feeds, it cannot be ruled out that any Market will enter a state of catastrophic failure at a point in the future (i.e. oracle used stops functioning and Market admin keys are compromised, so it cannot be changed), resulting in permanent inability to process closing positions and withdrawing collateral.

`BalancedVault` does not consider this case, exposing all funds deposited to a multi-market Vault to an increased risk, as it is not implementing a possibility for users to withdraw deposited funds through a partial emergency withdrawal from other markets, even at a price of losing the claim to locked funds in case it becomes available in the future. This risk is not mentioned in the documentation.

### Proof of Concept
Consider a Vault with 2 markets: ETH/USD and ARB/USD.

1. Alice deposits to Vault, her funds are split between 2 markets
2. ARB/USD market undergoes a fatal failure resulting in `maxAmount` returned from `_maxRedeemAtEpoch` to be 0
3. Alice cannot start withdrawal process as this line in `redeem` reverts:
```solidity
    if (shares.gt(_maxRedeemAtEpoch(context, accountContext, account))) revert BalancedVaultRedemptionLimitExceeded();
``` 

## Impact
Users funds are exposed to increased risk compared to depositing to each market individually and in case of failure of any of the markets all funds are lost. User has no possibility to consciously cut losses and withdraw funds from Markets other than the failed one.

## Code Snippet
`_maxRedeemAtEpoch`
https://github.com/sherlock-audit/2023-05-perennial/blob/0f73469508a4cd3d90b382eac2112f012a5a9852/perennial-mono/packages/perennial-vaults/contracts/balanced/BalancedVault.sol#L649-L677

check in `redeem`
https://github.com/sherlock-audit/2023-05-perennial/blob/0f73469508a4cd3d90b382eac2112f012a5a9852/perennial-mono/packages/perennial-vaults/contracts/balanced/BalancedVault.sol#L188

## Tool used

Manual Review

## Recommendation
Implement a partial/emergency withdrawal or acknowledge the risk clearly in Vault's documentation.