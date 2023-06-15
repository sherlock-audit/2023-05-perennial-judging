BLACK-PANDA-REACH

medium

# DoS when calling `_chargeFee` in `MultiInvoker`

## Summary
When calling `_chargeFee` function in `MultiInvoker` contract with an amount not divisible by `1e12` it will always result in a DoS. 

## Vulnerability Detail
In `MultiInvoker` contract, when pulling `USDC` from user to wrap it to `DSU`, the following syntax is used:

```solidity
USDC.pull(msg.sender, amount, true);
``` 

The third argument is to specify that `amount` has to be rounded up. For example, if `amount` is `1000000000000000001`, when converting it to 6 decimals it will be `1000001` instead of `1000000`. 

However, the newly implemented `_chargeFee` function is using the following syntax, forgetting about the third argument:

```solidity
function _chargeFee(address receiver, UFixed18 amount, bool wrapped) internal {
    if (wrapped) {
        USDC.pull(msg.sender, amount); // @audit-issue Not implementing third argument, DoS. 
        _handleWrap(receiver, amount);
    } else {
        USDC.pullTo(msg.sender, receiver, amount);
    }
}
```

The lack of the third argument causes the `amount` to be truncated from 18 decimals to 6. 

This is not the desired behaviour because later in the `_handleWrap` function it will always round up the amount if it's not divisible by `1e12`. Both `reserve.mint()` and `batcher.wrap()` will always round up the amount so if it's not rounded up in `_chargeFee`, it will revert later because of the lack of `USDC` funds. 

```solidity
function _handleWrap(address receiver, UFixed18 amount) internal {
    // If the batcher is 0 or  doesn't have enough for this wrap, go directly to the reserve
    if (address(batcher) == address(0) || amount.gt(DSU.balanceOf(address(batcher)))) {
        reserve.mint(amount);
        if (receiver != address(this)) DSU.push(receiver, amount);
    } else {
        // Wrap the USDC into DSU and return to the receiver
        batcher.wrap(amount, receiver);
    }
}
```

Imagine the following scenario:

1. Bob calls `_chargeFee` with an amount of `1000000000000100000` (or any amount not divisible by 1e12).
2. `MultiInvoker` converts `1000000000000100000` to `1e6` because it's not rounding up. 
3. `MultiInvoker` pulls `1e6` `USDC` from Bob. 
4. `MultiInvoker` calls `reserve.mint(amount)`, amount being `1000000000000100000`.
5. `Reserve` contract converts `1000000000000100000` to `1000001` because it's rounding up.
6. `Reserve` tries to pull `1000001` USDC from `MultiInvoker`.
7. Call reverts because `MultiInvoker` only owns `1e6` `USDC`. 

## Impact
This issue will cause the transaction to revert every time the `_chargeFunction` is called with an amount not divisible by `1e12` (`amount % 1e12 != 0`).

## Code Snippet
https://github.com/sherlock-audit/2023-05-perennial/blob/main/perennial-mono/packages/perennial/contracts/multiinvoker/MultiInvoker.sol#L379-L393

## Tool used

Manual Review

## Recommendation
In `_chargeFee` function, it's recommended to pull `USDC` tokens from caller using the following syntax:
```solidity
USDC.pull(msg.sender, amount, true);
```

This way the `USDC` amount received by the contract will be the same as the amount requested by the Batcher or Reserve contracts in `_handleWrap`, thus avoiding the DoS. 