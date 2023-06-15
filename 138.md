seeques

medium

# Liquidators will not receive any fees if account's newBalance drops below zero

## Summary
In the `liquidate()` function of the `Collateral` contract the total fee given to liquidator is calculated as account's total collateral or as the maintenance times liquidationFee, whichever is less. If during the settlement account's new balance becomes negative, the balance will be assigned to zero, so the fee amount for a liquidator would also be zero.

## Vulnerability Detail
Account's position of a specific product is updated during the settlement process. Within it, there is a `_settleAccount` function which calls the `settleAccount` function on the `Collateral` contract with the accumulated value for a given account:
```solidity
_controller.collateral().settleAccount(account, accumulated);
```
The `accumulated` parameter is based upon the global accumulator, account's current position and corresponding oracle version:
```solidity
    function syncTo(
        AccountAccumulator storage self,
        VersionedAccumulator storage global,
        AccountPosition storage position,
        uint256 versionTo
    ) internal returns (Accumulator memory value) {
        Accumulator memory valueAccumulated = global.valueAtVersion(versionTo)
            .sub(global.valueAtVersion(self.latestVersion));
        value = position.position.mul(valueAccumulated);
        self.latestVersion = versionTo;
    }
```
At the end of the settlement the account's balance settles as follows:
```solidity
    function settleAccount(OptimisticLedger storage self, address account, Fixed18 amount)
    internal returns (UFixed18 newShortfall) {
        Fixed18 newBalance = Fixed18Lib.from(self.balances[account]).add(amount);

        if (newBalance.sign() == -1) {
            newShortfall = newBalance.abs();
            newBalance = Fixed18Lib.ZERO;
        }

        self.balances[account] = newBalance.abs();
        self.shortfall = self.shortfall.add(newShortfall);
    }
```
So, in case when `accumulated` is negative and its modulus is more than of `self.balances[account]`, the new balance would be zero. This can happen when the position is leveraged too much and when the global value between versions becomes negative.

Now, the [`liquidate`](https://github.com/equilibria-xyz/perennial-mono/blob/b06d5145db62a312dd88dfcafef0f8e2166c5e32/packages/perennial/contracts/collateral/Collateral.sol#LL118C1-L118C1) function fetches total account's collateral from the same mapping the balance is updated during the settlement. It then calculates fees and pushes them to msg.sender:
```solidity
UFixed18 fee = UFixed18Lib.min(totalCollateral, collateralForFee.mul(liquidationFee));

_products[product].debitAccount(account, fee);
token.push(msg.sender, fee);
```
As we can see, if total collateral is zero liquidators will not receive any rewards.

## Impact
This is contrary to what is stated in the docs: 

*"Liquidators are immediately granted a liquidation reward equal to the maintenance requirement of the account times the `liquidationFee` upon the successful initiation of a liquidation."*

Furthermore, for such insolvent positions no third-party actor would be incentivised to perform liquidations. This may lead to unexpected behavior for product since such positions would impact product's state, though they should not.

## Code Snippet


## Tool used

Manual Review

## Recommendation
Ensure that liquidators get an appropriate fee or liquidate zero-collateralized positions automatically.
