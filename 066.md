roguereddwarf

medium

# BalancedVault.sol: Rebalancing logic can get stuck, leading to a loss of funds if a user wants to resolve it

## Summary
Each action that is executed in the Vault (`deposit`, `redeem`, `claim`, `sync`) calls the `_rebalance` function. This function first rebalances the collateral and then the position:
https://github.com/sherlock-audit/2023-05-perennial/blob/main/perennial-mono/packages/perennial-vaults/contracts/balanced/BalancedVault.sol#L431-L434

First collateral is withdrawn, then deposited (both if needed):
https://github.com/sherlock-audit/2023-05-perennial/blob/main/perennial-mono/packages/perennial-vaults/contracts/balanced/BalancedVault.sol#L440-L465

Then the position is reduced and increased (both if needed)
https://github.com/sherlock-audit/2023-05-perennial/blob/main/perennial-mono/packages/perennial-vaults/contracts/balanced/BalancedVault.sol#L499-L519

The problem is that rebalancing can revert which leaves the user that wants to execute an operation in the Vault no other choice other than sending additional assets to the Vault (note that calling the `deposit` function might not be possible because rebalancing reverts). The user needs to send the funds directly to the Vault, without calling the deposit function. Thereby this would be a loss of funds for the user because the additional funds would be shared across all users.

## Vulnerability Detail
Assume e.g. that the amount of collateral that the Vault has deposited into a Product is a little bit over the minimum collateral amount.

When the Product's makers are settled at a loss, the rebalancing logic in the Vault then tries to withdraw all collateral from the Product because it is below the minimum amount.

However this causes a revert because the position has not been closed yet and zero collateral cannot maintain the position.

## Impact
The Vault can get stuck and to resolve the situation, assets need to be sent to the Vault which are then essentially lost because they are shared among all users.

## Code Snippet
https://github.com/sherlock-audit/2023-05-perennial/blob/main/perennial-mono/packages/perennial-vaults/contracts/balanced/BalancedVault.sol#L431-L434

https://github.com/sherlock-audit/2023-05-perennial/blob/main/perennial-mono/packages/perennial-vaults/contracts/balanced/BalancedVault.sol#L440-L465

https://github.com/sherlock-audit/2023-05-perennial/blob/main/perennial-mono/packages/perennial-vaults/contracts/balanced/BalancedVault.sol#L499-L519

## Tool used
Manual Review

## Recommendation
It should not be possible that the rebalancing fails because it is required for all actions that can be executed in the Vault.
However solving the issue is not as simple as changing the order of actions in the call to `_rebalance` to this:
1. reduce position
2. reduce collateral
3. increase collateral
4. increase position

That's because there is delayed settlement in the Product and a change in position size is not immediately reflected.
Therefore a solution needs to be adopted that first reduces the position and then waits for settlement until steps 2-4 are executed.